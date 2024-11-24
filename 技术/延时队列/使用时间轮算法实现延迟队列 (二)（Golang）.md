#redis #延时队列 

在[上一篇文章中](https://raymondjiang.net/2020/12/10/implement-delay-queue-use-golang/)，我们实现了一个简单的延迟队列，但也遗留下一些问题没有解决；最大的问题就是如果程序中断后重启，哪些尚未执行的任务将永远丢失无法执行，对于一个生产级别的程序来说，这是无法容忍的。今天我们就通过为延迟队列添加持久化功能，来解决这个问题。

-   创建一个持久化接口，在延迟队列初始化时传入，延迟队列通过该接口，对任务进行持久化操作。接口的使用会让我们的延迟队列可扩展性更高，用户可以实现自己的持久化方案来构建一个稳定可靠的持久化队列。

```go
type Persistence interface {
    Save(task *Task) error
    GetList() []*Task
    Delete(taskId string) error
}
```

为了在用户没有提供持久化对象时，延迟队列仍然可以运行，我们为该持久化接口提供一个默认实现，这里选用 redis 作为底层组件。

```go
var lock sync.Once

const (
    //每个task存储到redis当中的key前缀，便于与其他数据的区分度
    TASK_KEY_PREFIX = "delaytk_"
)

var redisInstance *redisDb

type redisDb struct {
    Client *redis.Client
    //task 列表key，该redis列表用于存储task的任务ID
    TaskListKey string
}

//单列方法
func getRedisDb() *redisDb {

    lock.Do(func() {
        dbNumber, _ := strconv.Atoi(GetEvnWithDefaultVal("REDIS_DB", "0"))
        redisInstance = &redisDb{
            Client: redis.NewClient(&redis.Options{
                Addr:     GetEvnWithDefaultVal("REDIS_ADDR", "localhost:6379"),
                Password: GetEvnWithDefaultVal("REDIS_PWD", ""), // no password set
                DB:       int64(dbNumber),                       // use  DB
            }),
            TaskListKey: GetEvnWithDefaultVal("DELAY_QUEUE_LIST_KEY", "__delay_queue_list__"),
        }
    })

    return redisInstance
}

//将task保存到redis, 将task id 存入 list, task 整体内容放入其 id 对应数据槽
func (rd *redisDb) Save(task *Task) error {
    tk, err := json.Marshal(task)
    if err != nil {
        log.Println(err)
        return err
    }
    if string(tk) != "" {
        key := fmt.Sprintf("%s%s", TASK_KEY_PREFIX, task.Id)
        //如果key不存在
        if val, _ := rd.Client.Get(key).Result(); val == "" {
            rd.Client.LPush(rd.TaskListKey, task.Id)
        }
        result := rd.Client.Set(key, string(tk), 0)
        return result.Err()

    } else {
        return errors.New("task is emtpy")
    }

}

//从 redis 中恢复持久化的 task 列表
func (rd *redisDb) GetList() []*Task {
    listResult := rd.Client.LRange(rd.TaskListKey, 0, -1)
    listArray, _ := listResult.Result()
    tasks := []*Task{}
    if listArray != nil && len(listArray) > 0 {
        for _, item := range listArray {
            key := fmt.Sprintf("%s%s", TASK_KEY_PREFIX, item)
            taskCmd := rd.Client.Get(key)
            if val, err := taskCmd.Result(); err == nil {
                entity := Task{}
                err := json.Unmarshal([]byte(val), &entity)
                if err == nil {
                    tasks = append(tasks, &entity)
                }
            }
        }
    }
    return tasks
}

//从 redis 从删除某个 task
func (rd *redisDb) Delete(taskId string) error {
    rd.Client.LRem(rd.TaskListKey, 0, taskId)
    rd.Client.Del(fmt.Sprintf("%s%s", TASK_KEY_PREFIX, taskId))

    return nil
}

//从 redis 中清空所有 task
func (rd *redisDb) RemoveAll() error {
    listResult := rd.Client.LRange(rd.TaskListKey, 0, -1)
    listArray, _ := listResult.Result()
    if listArray != nil && len(listArray) > 0 {
        for _, tkId := range listArray {
            rd.Client.Del(fmt.Sprintf("%s%s", TASK_KEY_PREFIX, tkId))
        }
    }
    rd.Client.Del(rd.TaskListKey)
    return nil
}
```

-   在上一篇文章中，我们是将任务工厂指针（TaskExecutor）存储到task中，为的是不同的任务可以拥有不同的任务构建工厂，但是这里会带来一个问题，就是从持久存储中恢复对象时，golang反射限制（无法通过字符串反射构建对象），导致我们无法获取真实的调用方法；因此我们移除该字段，将它放到 delayQueue 中，用户在构建延迟队列时，需要将具体的工厂方法传入并对其进行初始化。 因此我们修改了，task , delayQueue 的部分实现代码，现在它变成这样：

```go
type Task struct {
    //任务ID便于持久化
    Id string
    //任务在时间轮上的循环次数，等于0时，执行该任务
    CycleCount int
    //任务在时间轮上的位置
    WheelPosition int
    //任务类型，用于工厂方法判断该使用哪个实现对象
    TaskType string
    //任务方法参数
    TaskParams string

    Next *Task
}
```

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

为了可以接收外部传入的环境变量，我们创建了一个可以带默认值的通用方法：

```go
func GetEvnWithDefaultVal(key string, defaultVal string) string {
    val := os.Getenv(key)
    if val != "" {
        return val
    } else {
        return defaultVal
    }
}
```

现在重新修改测试用例，对延迟队列进行测试：

```go
func TestRunDelayQueue(t *testing.T) {
    c := make(chan struct{})
    // redis := getRedisDb()
    // redis.RemoveAll()

    queue := GetDelayQueue(commonFactory)
    queue.Start()

    go func() {
        //Simulate pushing tasks to the time wheel
        q := GetDelayQueue(commonFactory)
        q.Push(5, "RetryNotify", `{name: "raymond"}`)
        q.Push(15, "GoodComments", `{name: "raymond"}`)
        q.Push(25, "TaskOne", `{name: "raymond"}`)
        q.Push(35, "TaskThree", `{name: "raymond"}`)
        q.Push(45, "TaskFour", `{name: "raymond"}`)

    }()

    c <- struct{}{}
}
```

在延迟队列运行时随意中断程序，然后重新运行程序，之前未运行的任务，仍然可以继续执行。

最后，我们终于拥有了一个可持久化，并从中断中恢复任务的持久化队列。它应该可以在生产环境中进行测试了吧：）

# Reference 
https://raymondjiang.net/2020/12/20/implement-delay-queue-use-golang-2/
https://github.com/0RaymondJiang0/go-delayqueue
