#redis #延时队列

实际项目当中，经常会使用延迟队列，执行一些延迟任务，比如：

-   如果第一次通知失败，就延迟 X 分钟再通知一次，这在构建webhook的项目中很常见。
-   如果用户成功消费，X 天后没有评论，就默认给予好评。

这些任务具有的共同点可以抽象为：在某项任务执行完毕后，延迟一定时间执行另外一项任务。由于这些任务分布在随机的时间点上，构建定时JOB去跑这些任务必定是一件非常繁琐的工程。即使采用每次计算好执行时间，然后存储到缓存或者DB里面，仍然面临每次需要遍历庞大任务数据，并且执行时间判断造成的误差会导致某些任务也许无法执行的窘境。那么一个设计优良的延迟队列，为了解决上述问题，应该具备哪些特性呢？我觉得，它应该具备以下特性：

-   延迟时间要精准，保证在延迟到达时，执行预先设定的任务。
-   任务的执行时间，不应太过于影响时间计时器的运行。
-   延迟粒度可以自由配置，时间间隔可大可小。

要实现延迟队列，目前比较常用的方式，是使用时间轮算法，该算法的结构大体如下图所示：

![](https://raymondjiang.net/images/timewheel1.png)

它由三部分组成：

-   一个计时器，或者称为时间发生器。
-   一个循环队列，一般通过算法由数组来实现。
-   一个存储任务（task）的数据结构，可以是数组或者链表。

其中前两个决定了延迟的粒度，最后一个决定任务的存储方式；当任务执行完毕后，需要将任务从队列中删除，因此采用链表来存储任务，应该效率要比数组高很多（因为整个任务执行过程，不存在随机读取任务的需求）。

现在我们就用go语言来实现一个相对通用的延迟队列：

1.  先来构建一个简单任务执行接口，具体的业务对象可以实现它来执行任务。

```go
type Executor interface {
    DoDelayTask(contents string) error
}  
```

2.  然后构建存储在时间轮中的任务，每个任务节点，我们采用链表实现。

```go

//定义一个获取实现具体任务对象的工厂方法，该方法需要在业务项目中定义并实现
type BuildExecutor func(taskType string) Executor

//任务采用链表结构
type Task struct {
    //任务在时间轮上的循环次数，等于0时，执行该任务
    CycleCount   int
    //任务类型，用于工厂方法判断该使用哪个实现对象
    TaskType     string
    //任务方法参数
    TaskParams   string
    //任务工厂方法指针
    TaskExecutor BuildExecutor
    Next         *Task
}

//通过工厂方法获取具体实现，然后调用方法，执行任务
func (t *Task) Execute() error {
    if t.TaskExecutor != nil {
        executor := t.TaskExecutor(t.TaskType)
        if executor != nil {
            return executor.DoDelayTask(t.TaskParams)
        } else {
            return errors.New("executor is nil")
        }
    } else {
        return errors.New("task build executor is nil")
    }

}
```

3.  最后，用时间轮算法实现时间轮，并将任务通知与延迟队列结合。

```go
const (
    //轮的时间长度，目前设置为一个小时，也就是时间轮每循环一次需要1小时；默认时间轮上面的每走一步的最小粒度为1秒。
    WHEEL_SIZE = 3600
)

//定义一个获取实现具体任务对象的工厂方法，该方法需要在业务项目中定义并实现
type BuildExecutor func(taskType string) Executor

var onceNew sync.Once
var onceStart sync.Once

var delayQueueInstance *delayQueue

type wheel struct {
    //所有在时间轮上的任务均采用链表形式存储
    //如果采用数组，对应已经执行过的任务，会造成不必要的空间浪费或者数组移动造成的时间复杂度
    NotifyTasks *Task
}

type delayQueue struct {
    //循环队列
    TimeWheel    [WHEEL_SIZE]wheel
    CurrentIndex uint //时间轮当前指针
    Persistence
    //任务工厂方法指针, 需要在对象创建时初始化它
    TaskExecutor BuildExecutor
}

//单列方法,使用默认的redis持久方案
func GetDelayQueue(serviceBuilder BuildExecutor) *delayQueue {
    //保证只初始化一次
    onceNew.Do(func() {
        delayQueueInstance = &delayQueue{
            Persistence:  getRedisDb(),
            TaskExecutor: serviceBuilder,
        }
    })
    return delayQueueInstance
}

//单列方法，使用外部传入的持久方案
func GetDelayQueueWithPersis(serviceBuilder BuildExecutor, persistence Persistence) *delayQueue {
    if persistence == nil {
        log.Fatalf("persistance is null")
    }
    //保证只初始化一次
    onceNew.Do(func() {
        delayQueueInstance = &delayQueue{
            Persistence:  persistence,
            TaskExecutor: serviceBuilder,
        }
    })
    return delayQueueInstance
}

func (dq *delayQueue) Start() {
    //保证只会有一个时间轮计时器
    onceStart.Do(dq.init)
}

func (dq *delayQueue) init() {
    go func() {
        //从缓存中加载持久化的任务
        dq.loadTasksFromDb()
        for {
            select {
            //默认时间轮上的最小粒度为1秒
            case <-time.After(time.Second * 1):

                if dq.CurrentIndex >= WHEEL_SIZE {
                    dq.CurrentIndex = dq.CurrentIndex % WHEEL_SIZE
                }

                taskLinkHead := dq.TimeWheel[dq.CurrentIndex].NotifyTasks
                //遍历链表
                //当前节点前一指针
                prev := taskLinkHead
                //当前节点指针
                p := taskLinkHead
                for p != nil {
                    if p.CycleCount == 0 {
                        taskId := p.Id
                        //开启新的go routing 去做通知，加快每次遍历的速度，确保不会拖慢时间轮的运行
                        //如果任务有异常，尽量让具体的业务对象去处理，延迟队列不处理具体业务异常，
                        //这样可以保证延迟队列的业务单纯性，避免难以维护的问题。如果具体业务出现问题，需要重复通知，可以将任务重新加入队列即可。
                        go dq.ExecuteTask(p.TaskType, p.TaskParams)
                        //删除链表节点 task
                        //如果是第一个节点
                        if prev == p {
                            dq.TimeWheel[dq.CurrentIndex].NotifyTasks = p.Next
                            prev = p.Next
                            p = p.Next
                        } else {
                            //如果不是第一个节点
                            prev.Next = p.Next
                            p = p.Next
                        }
                        //从持久对象上删除该任务
                        dq.Persistence.Delete(taskId)

                    } else {
                        p.CycleCount--
                        prev = p
                        p = p.Next
                    }

                }

                dq.CurrentIndex++

            }
        }
    }()
}

func (dq *delayQueue) loadTasksFromDb() {
    tasks := dq.Persistence.GetList()
    if tasks != nil && len(tasks) > 0 {
        for _, task := range tasks {
            // fmt.Printf("%v\n", task)
            delaySeconds := ((task.CycleCount + 1) * WHEEL_SIZE) + task.WheelPosition
            if delaySeconds > 0 {
                dq.internalPush(time.Duration(delaySeconds)*time.Second, task.Id, task.TaskType, task.TaskParams, false)
            }
        }
    }
}

//将任务加入延迟队列
func (dq *delayQueue) Push(delaySeconds time.Duration, taskType string, taskParams interface{}) error {

    var pms string
    result, ok := taskParams.(string)
    if !ok {
        tp, _ := json.Marshal(taskParams)
        pms = string(tp)
    } else {
        pms = result
    }

    return dq.internalPush(delaySeconds, "", taskType, pms, true)
}

func (dq *delayQueue) internalPush(delaySeconds time.Duration, taskId string, taskType string, taskParams string, notNeedPresis bool) error {
    if int(delaySeconds.Seconds()) == 0 {
        errorMsg := fmt.Sprintf("the delay time cannot be less than 1 second, current is: %v", delaySeconds)
        log.Println(errorMsg)
        return errors.New(errorMsg)
    }
    //从当前时间指针处开始计时
    calculateValue := int(dq.CurrentIndex) + int(delaySeconds.Seconds())

    cycle := calculateValue / WHEEL_SIZE
    if cycle > 0 {
        cycle--
    }
    index := calculateValue % WHEEL_SIZE

    if taskId == "" {
        u := uuid.New()
        taskId = u.String()
    }
    task := &Task{
        Id:            taskId,
        CycleCount:    cycle,
        WheelPosition: index,
        TaskType:      taskType,
        TaskParams:    taskParams,
    }
    if dq.TimeWheel[index].NotifyTasks == nil {
        dq.TimeWheel[index].NotifyTasks = task
        // log.Println(dq.TimeWheel[index].NotifyTasks)
    } else {
        //将新任务插入链表头，由于任务之间没有顺序关系，这种实现最为简单
        head := dq.TimeWheel[index].NotifyTasks
        task.Next = head
        dq.TimeWheel[index].NotifyTasks = task
        // log.Println(dq.TimeWheel[index].NotifyTasks)
    }
    if notNeedPresis {
        //持久化任务
        dq.Persistence.Save(task)
    }

    return nil
}

//通过工厂方法获取具体实现，然后调用方法，执行任务
func (dq *delayQueue) ExecuteTask(taskType, taskParams string) error {
    if dq.TaskExecutor != nil {
        executor := dq.TaskExecutor(taskType)
        if executor != nil {
            log.Printf("Execute task: %s with params: %s\n", taskType, taskParams)

            return executor.DoDelayTask(taskParams)
        } else {
            return errors.New("executor is nil")
        }
    } else {
        return errors.New("task build executor is nil")
    }

}

//延迟队列上，某个时间轮上的任务数量
func (dq *delayQueue) WheelTaskQuantity(index int) int {
    tasks := dq.TimeWheel[index].NotifyTasks
    if tasks == nil {
        return 0
    }
    k := 0
    for p := tasks; p != nil; p = p.Next {
        k++
    }

    return k
}
```

上面我们用单列模式来构建延迟队列，因为很多时候，我们程序当中需要保证任何时刻只能有一个时间轮队列存在，不然会造成资源浪费。我们设置时间轮的大小和最小粒度分别为一个小时和一秒钟，这也是最为常用的一种方式。 用一个1秒的时间发生器配合select来驱动时间轮的运行，在每个轮节点上，遍历链表，查找需要执行的任务；任务执行时，无须处理异常，让具体业务对象去处理业务异常，保证时间轮的单纯性。  
最后，我们写一个测试用例来测试时间轮的运行：

```go
//创建一个测试业务对象，让它实现 Executor 接口   
type businessNotify struct {
}

func (bn *businessNotify) DoDelayTask(contents string) error {
    fmt.Println(fmt.Sprintf("Do task.....%s", contents))
    return nil
}

//一个工厂方法，通过类型去构建任务执行对象
func commonFactory(taskType string) Executor {
    //do filter business service by taskType ...
    //...
    return &businessNotify{}
}
//测试任务的执行
func TestTaskExecute(t *testing.T) {
    task := &Task{
        CycleCount:   1,
        TaskType:     "RetryNotify",
        TaskParams:   `{name: "raymond"}`,
        TaskExecutor: commonFactory,
    }
    task.Execute()
}

//测试延迟队列的运行
func TestRunDelayQueue(t *testing.T) {
    c := make(chan struct{})

    queue := GetDelayQueue()
    queue.Start()

    go func() {
        //模拟在其他地方将数据压入延迟队列
        q := GetDelayQueue()
        q.Push(5, "RetryNotify", `{name: "raymond"}`, commonFactory)
        q.Push(15, "GoodComments", `{name: "raymond"}`, commonFactory)
        q.Push(25, "TaskOne", `{name: "raymond"}`, commonFactory)
        q.Push(35, "TaskThree", `{name: "raymond"}`, commonFactory)
        q.Push(45, "TaskFour", `{name: "raymond"}`, commonFactory)

    }()

    c <- struct{}{}
}
```

现在各方面无误的话，可以看到时间轮正常运行，并在合适的时间执行对应的任务。

按理说一个简单的延迟队列就算实现完毕了，是不是我们就可以把它 run on production？且慢。。。一个真正可以在生产环境当中运行的延迟队列，还有很多事情需要考虑，比如：

-   任务是否需要持久化？如果不持久化，如何保证程序挂掉后，未执行的任务可以恢复执行？
-   采用何种持久化方案，更能适配当前的业务？
-   任务出错后，该采用什么补救方案（比如重试），才能保证任务执行的可靠性。

下次，我们将慢慢细化该延迟队列，将它打造成一个可以在生产环境稳定运行的程序 。。。
