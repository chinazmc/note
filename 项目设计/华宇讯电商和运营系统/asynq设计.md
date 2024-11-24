# 0x00 前言 

[asynq](https://github.com/hibiken/asynq)：Golang distributed task queue library，许多设计思想都来自[sidekiq](https://github.com/mperham/sidekiq)。[官方文档](https://github.com/hibiken/asynq/wiki)给出的特点如下（省略了若干）：

- 任务已写入Redis后会持久化（支持Redis Cluster/Sentinels）
- 任务执行失败自动 retry
- 支持任务优先级权重（加权优先级队列）
- 支持定时发送任务
- 可使用 unique-option 來避免任务重复执行
- UI及客户端、metrics支持（CLI检查和远程控制队列和任务）
- 支持任务设置执行时间or最长可运行的时间

库的使用示例[在此](https://github.com/hibiken/asynq/wiki/Getting-Started)，本文基于`v0.23.0`[版本](https://github.com/hibiken/asynq/releases/tag/v0.23.0)

#### asynq架构 

![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/asyncq/asyncq-arch.jpg)

## 0x01 代码分析 

asynq整体流程上和先前分析的中间件大同小异，并且该中间件严重依赖Redis，所有的原子化逻辑都是通过lua脚本实现的（如任务的出入队列等等），按照任务的状态划分了不同的redis数据结构及[逻辑](https://github.com/hibiken/asynq/blob/master/internal/rdb/rdb.go)：

- active：asynq:{}:active LIST类型
- pending：asynq:{}:pending LIST类型
- lease：asynq:{}:lease SortedSet类型
- completed：asynq:{}: SortedSet类型
- paused：asynq:{}:paused：HASHTABLE类型

此外，还有各种不同类型的结构，用来记录任务执行的状态、计数器等等
![[Pasted image 20240220144753.png]]

#### 任务流

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/asyncq/asyncq-flow.png)

#### 任务操作的Redis封装 

1、[`dequeueCmd`](https://github.com/hibiken/asynq/blob/master/internal/rdb/rdb.go#L204)：用于把任务从`pending`List取出，放到`active`List中

```go
// Input:
// KEYS[1] -> asynq:{<qname>}:pending
// KEYS[2] -> asynq:{<qname>}:paused
// KEYS[3] -> asynq:{<qname>}:active
// KEYS[4] -> asynq:{<qname>}:lease
// --
// ARGV[1] -> initial lease expiration Unix time
// ARGV[2] -> task key prefix
//
// Output:
// Returns nil if no processable task is found in the given queue.
// Returns an encoded TaskMessage.
//
// Note: dequeueCmd checks whether a queue is paused first, before
// calling RPOPLPUSH to pop a task from the queue.
var dequeueCmd = redis.NewScript(`
if redis.call("EXISTS", KEYS[2]) == 0 then
	local id = redis.call("RPOPLPUSH", KEYS[1], KEYS[3])
	if id then
		local key = ARGV[2] .. id
		redis.call("HSET", key, "state", "active")
		redis.call("HDEL", key, "pending_since")
		redis.call("ZADD", KEYS[4], ARGV[1], id)
		return redis.call("HGET", key, "msg")
	end
end
return nil`)
```

2、`markAsCompleteCmd`方法：任务成功处理完成，通过`HSET`更新状态，同时删除`active`LIST的数据

```Go
// KEYS[1] -> asynq:{<qname>}:active
// KEYS[2] -> asynq:{<qname>}:lease
// KEYS[3] -> asynq:{<qname>}:completed
// KEYS[4] -> asynq:{<qname>}:t:<task_id>
// KEYS[5] -> asynq:{<qname>}:processed:<yyyy-mm-dd>
// KEYS[6] -> asynq:{<qname>}:processed
//
// ARGV[1] -> task ID
// ARGV[2] -> stats expiration timestamp
// ARGV[3] -> task exipration time in unix time
// ARGV[4] -> task message data
// ARGV[5] -> max int64 value
var markAsCompleteCmd = redis.NewScript(`
if redis.call("LREM", KEYS[1], 0, ARGV[1]) == 0 then
  return redis.error_reply("NOT FOUND")
end
if redis.call("ZREM", KEYS[2], ARGV[1]) == 0 then
  return redis.error_reply("NOT FOUND")
end
if redis.call("ZADD", KEYS[3], ARGV[3], ARGV[1]) ~= 1 then
  redis.redis.error_reply("INTERNAL")
end
redis.call("HSET", KEYS[4], "msg", ARGV[4], "state", "completed")
local n = redis.call("INCR", KEYS[5])
if tonumber(n) == 1 then
	redis.call("EXPIREAT", KEYS[5], ARGV[2])
end
local total = redis.call("GET", KEYS[6])
if tonumber(total) == tonumber(ARGV[5]) then
	redis.call("SET", KEYS[6], 1)
else
	redis.call("INCR", KEYS[6])
end
return redis.status_reply("OK")
`)
```

#### 核心结构 

1、`processor`结构  

```Go
type processor struct {
	logger *log.Logger
	broker base.Broker
	clock  timeutil.Clock

	handler   Handler
	baseCtxFn func() context.Context

	queueConfig map[string]int

	// orderedQueues is set only in strict-priority mode.
	orderedQueues []string

	retryDelayFunc RetryDelayFunc
	isFailureFunc  func(error) bool

	errHandler ErrorHandler

	shutdownTimeout time.Duration

	// channel via which to send sync requests to syncer.
	syncRequestCh chan<- *syncRequest

	// rate limiter to prevent spamming logs with a bunch of errors.
	errLogLimiter *rate.Limiter

	// sema is a counting semaphore to ensure the number of active workers
	// does not exceed the limit.
	sema chan struct{}

	// channel to communicate back to the long running "processor" goroutine.
	// once is used to send value to the channel only once.
	done chan struct{}
	once sync.Once

	// quit channel is closed when the shutdown of the "processor" goroutine starts.
	quit chan struct{}

	// abort channel communicates to the in-flight worker goroutines to stop.
	abort chan struct{}

	// cancelations is a set of cancel functions for all active tasks.
	cancelations *base.Cancelations

	starting chan<- *workerInfo
	finished chan<- *base.TaskMessage
}

```

#### 任务的状态 

每个任务的唯一key`asynq:{<qname>}:t:<task_id>`，在Redis数据结构为HashTable，有如下字段：

- `msg`：任务关联数据
- `state`：任务状态（`pending`、`active`、`completed`、`aggregating`、`scheduled`、`retry`、`archived`）
- `pending_since`：
- `unique_key`：
- `group`：
- `result`：

1、[`Enqueue`](https://github.com/hibiken/asynq/blob/master/internal/rdb/rdb.go#L96)方法，关联`enqueueCmd`  

```Go
// enqueueCmd enqueues a given task message.
//
// Input:
// KEYS[1] -> asynq:{<qname>}:t:<task_id>
// KEYS[2] -> asynq:{<qname>}:pending
// --
// ARGV[1] -> task message data
// ARGV[2] -> task ID
// ARGV[3] -> current unix time in nsec
//
// Output:
// Returns 1 if successfully enqueued
// Returns 0 if task ID already exists
var enqueueCmd = redis.NewScript(`
if redis.call("EXISTS", KEYS[1]) == 1 then
	return 0
end
redis.call("HSET", KEYS[1],
           "msg", ARGV[1],
           "state", "pending",
           "pending_since", ARGV[3])
redis.call("LPUSH", KEYS[2], ARGV[2])
return 1
`)
```

2、`enqueueUniqueCmd`  

```Go
// enqueueUniqueCmd enqueues the task message if the task is unique.
//
// KEYS[1] -> unique key
// KEYS[2] -> asynq:{<qname>}:t:<taskid>
// KEYS[3] -> asynq:{<qname>}:pending
// --
// ARGV[1] -> task ID
// ARGV[2] -> uniqueness lock TTL
// ARGV[3] -> task message data
// ARGV[4] -> current unix time in nsec
//
// Output:
// Returns 1 if successfully enqueued
// Returns 0 if task ID conflicts with another task
// Returns -1 if task unique key already exists
var enqueueUniqueCmd = redis.NewScript(`
local ok = redis.call("SET", KEYS[1], ARGV[1], "NX", "EX", ARGV[2])
if not ok then
  return -1 
end
if redis.call("EXISTS", KEYS[2]) == 1 then
  return 0
end
redis.call("HSET", KEYS[2],
           "msg", ARGV[3],
           "state", "pending",
           "pending_since", ARGV[4],
           "unique_key", KEYS[1])
redis.call("LPUSH", KEYS[3], ARGV[1])
return 1
`)
```

3、`dequeueCmd`  

```Go

// Input:
// KEYS[1] -> asynq:{<qname>}:pending
// KEYS[2] -> asynq:{<qname>}:paused
// KEYS[3] -> asynq:{<qname>}:active
// KEYS[4] -> asynq:{<qname>}:lease
// --
// ARGV[1] -> initial lease expiration Unix time
// ARGV[2] -> task key prefix
//
// Output:
// Returns nil if no processable task is found in the given queue.
// Returns an encoded TaskMessage.
//
// Note: dequeueCmd checks whether a queue is paused first, before
// calling RPOPLPUSH to pop a task from the queue.
var dequeueCmd = redis.NewScript(`
if redis.call("EXISTS", KEYS[2]) == 0 then
	local id = redis.call("RPOPLPUSH", KEYS[1], KEYS[3])
	if id then
		local key = ARGV[2] .. id
		redis.call("HSET", key, "state", "active")
		redis.call("HDEL", key, "pending_since")
		redis.call("ZADD", KEYS[4], ARGV[1], id)
		return redis.call("HGET", key, "msg")
	end
end
return nil`)
```

举例来说，redis数据如下：

```bash
127.0.0.1:6379[1]> keys *
 1) "asynq:servers:{VM_120_245_centos:20022:69678348-e2e8-4a40-a729-6f50b8180194}"
 2) "asynq:{critical}:processed:2022-10-17"
 3) "asynq:servers"
 4) "asynq:{low}:scheduled"
 5) "asynq:{low}:processed:2022-10-17"
 6) "asynq:workers"
 7) "asynq:{critical}:processed:2022-10-08"
 8) "dq_queue_order"
 9) "asynq:{low}:processed:2022-10-08"
10) "1570239831121"
11) "15702398321"
12) "asynq:queues"
13) "asynq:{default}:processed:2022-10-08"
```

#### 核心流程 

1、`processor.exec`  
asynq没有使用 `brpop/rpop` 指令（前者阻塞，后者轮询），而是通过 channel 实现信号量去控制对 Redis 的轮询。参见下面的`processor.exec`方法，该方法独立运行，会先尝试获取一个允许继续执行的信号，如果有则调用 broker的`Dequeue` 方法查询待执行的任务信息，否则就停下来等待信号。如果队列是空的，那么会`time.Sleep(time.Second)`，以免不断的查询 Redis 。此外，执行信号的个数也是可配置（默认机器核数），这里就是普通的限制取任务并发

```Go
// exec pulls a task out of the queue and starts a worker goroutine to
// process the task.
func (p *processor) exec() {
	select {
	case <-p.quit:
		return
	case p.sema <- struct{}{}: // acquire token
		qnames := p.queues()

        //从broker中取任务数据
		msg, leaseExpirationTime, err := p.broker.Dequeue(qnames...)
		switch {
		case errors.Is(err, errors.ErrNoProcessableTask):
			p.logger.Debug("All queues are empty")
			// Queues are empty, this is a normal behavior.
			// Sleep to avoid slamming redis and let scheduler move tasks into queues.
			// Note: We are not using blocking pop operation and polling queues instead.
			// This adds significant load to redis.
			time.Sleep(time.Second)
			<-p.sema // release token
			return
		case err != nil:
			if p.errLogLimiter.Allow() {
				p.logger.Errorf("Dequeue error: %v", err)
			}
			<-p.sema // release token
			return
		}

		lease := base.NewLease(leaseExpirationTime)
		deadline := p.computeDeadline(msg)
		p.starting <- &workerInfo{msg, time.Now(), deadline, lease}
		go func() {
			defer func() {
				p.finished <- msg
				<-p.sema // release token
			}()

			ctx, cancel := asynqcontext.New(p.baseCtxFn(), msg, deadline)
			p.cancelations.Add(msg.ID, cancel)
			defer func() {
				cancel()
				p.cancelations.Delete(msg.ID)
			}()

			// check context before starting a worker goroutine.
			select {
			case <-ctx.Done():
				// already canceled (e.g. deadline exceeded).
				p.handleFailedMessage(ctx, lease, msg, ctx.Err())
				return
			default:
			}

			resCh := make(chan error, 1)
			go func() {
				task := newTask(
					msg.Type,
					msg.Payload,
					&ResultWriter{
						id:     msg.ID,
						qname:  msg.Queue,
						broker: p.broker,
						ctx:    ctx,
					},
				)
				resCh <- p.perform(ctx, task)
			}()

			select {
			case <-p.abort:
				// time is up, push the message back to queue and quit this worker goroutine.
				p.logger.Warnf("Quitting worker. task id=%s", msg.ID)
				p.requeue(lease, msg)
				return
			case <-lease.Done():
				cancel()
				p.handleFailedMessage(ctx, lease, msg, ErrLeaseExpired)
				return
			case <-ctx.Done():
				p.handleFailedMessage(ctx, lease, msg, ctx.Err())
				return
			case resErr := <-resCh:
				if resErr != nil {
					p.handleFailedMessage(ctx, lease, msg, resErr)
					return
				}
				p.handleSucceededMessage(lease, msg)
			}
		}()
	}
}
```

2、任务执行  
任务的执行也是异步的

```Go
func (p *processor) exec() {
    //...
    go func() {
                    task := newTask(
                        msg.Type,
                        msg.Payload,
                        &ResultWriter{
                            id:     msg.ID,
                            qname:  msg.Queue,
                            broker: p.broker,
                            ctx:    ctx,
                        },
                    )
                    resCh <- p.perform(ctx, task)
                }()

    //...
}
// perform calls the handler with the given task.
// If the call returns without panic, it simply returns the value,
// otherwise, it recovers from panic and returns an error.
func (p *processor) perform(ctx context.Context, task *Task) (err error) {
	defer func() {
		if x := recover(); x != nil {
			p.logger.Errorf("recovering from panic. See the stack trace below for details:\n%s", string(debug.Stack()))
			_, file, line, ok := runtime.Caller(1) // skip the first frame (panic itself)
			if ok && strings.Contains(file, "runtime/") {
				// The panic came from the runtime, most likely due to incorrect
				// map/slice usage. The parent frame should have the real trigger.
				_, file, line, ok = runtime.Caller(2)
			}

			// Include the file and line number info in the error, if runtime.Caller returned ok.
			if ok {
				err = fmt.Errorf("panic [%s:%d]: %v", file, line, x)
			} else {
				err = fmt.Errorf("panic: %v", x)
			}
		}
	}()
	return p.handler.ProcessTask(ctx, task)
}

func (fn HandlerFunc) ProcessTask(ctx context.Context, task *Task) error {
	return fn(ctx, task)
}
```

3、等待任务执行结果  
开头列举的，asynq支持任务的控制核心就在此实现，如；

- `p.abort`：
- `lease.Done()`：任务执行过期（超时）了
- `ctx.Done()`：
- `<-resCh`：任务正常结束，获取结果成功/失败

```GOLANG
func (p *processor) exec() {
    //...
    select {
			case <-p.abort:
				// time is up, push the message back to queue and quit this worker goroutine.
				p.logger.Warnf("Quitting worker. task id=%s", msg.ID)
				p.requeue(lease, msg)
				return
			case <-lease.Done():
				cancel()
				p.handleFailedMessage(ctx, lease, msg, ErrLeaseExpired)
				return
			case <-ctx.Done():
				p.handleFailedMessage(ctx, lease, msg, ctx.Err())
				return
			case resErr := <-resCh:
				if resErr != nil {
					p.handleFailedMessage(ctx, lease, msg, resErr)
					return
				}
				p.handleSucceededMessage(lease, msg)
			}
    //...
}

//处理结果
func (p *processor) handleSucceededMessage(l *base.Lease, msg *base.TaskMessage) {
	if msg.Retention > 0 {
		//
		p.markAsComplete(l, msg)
	} else {
		p.markAsDone(l, msg)
	}
}
```

注意：`markAsComplete`是

4、处理任务执行失败：`handleFailedMessage`  
最终，需要retry重试的任务在`p.retry`方法中被打上下一次重试的时间，然后在`retryCmd`中通过`ZADD`添加到`asynq:{<qname>}:retry`对应的SortedSet中

```Go
func (p *processor) handleFailedMessage(ctx context.Context, l *base.Lease, msg *base.TaskMessage, err error) {
	if p.errHandler != nil {
		p.errHandler.HandleError(ctx, NewTask(msg.Type, msg.Payload), err)
	}
	if !p.isFailureFunc(err) {
		// retry the task without marking it as failed
		p.retry(l, msg, err, false /*isFailure*/)
		return
	}
	if msg.Retried >= msg.Retry || errors.Is(err, SkipRetry) {
		p.logger.Warnf("Retry exhausted for task id=%s", msg.ID)
		p.archive(l, msg, err)
	} else {
		//调用p.retry重试
		p.retry(l, msg, err, true /*isFailure*/)
	}
}

func (p *processor) retry(l *base.Lease, msg *base.TaskMessage, e error, isFailure bool) {
	if !l.IsValid() {
		// If lease is not valid, do not write to redis; Let recoverer take care of it.
		return
	}
	ctx, _ := context.WithDeadline(context.Background(), l.Deadline())

	//计算下一次重试触发的时间
	d := p.retryDelayFunc(msg.Retried, e, NewTask(msg.Type, msg.Payload))
	retryAt := time.Now().Add(d)

	//调用broker的retry的方法
	err := p.broker.Retry(ctx, msg, retryAt, e.Error(), isFailure)
	if err != nil {
		errMsg := fmt.Sprintf("Could not move task id=%s from %q to %q", msg.ID, base.ActiveKey(msg.Queue), base.RetryKey(msg.Queue))
		p.logger.Warnf("%s; Will retry syncing", errMsg)
		p.syncRequestCh <- &syncRequest{
			fn: func() error {
				return p.broker.Retry(ctx, msg, retryAt, e.Error(), isFailure)
			},
			errMsg:   errMsg,
			deadline: l.Deadline(),
		}
	}
}

// Retry moves the task from active to retry queue.
// It also annotates the message with the given error message and
// if isFailure is true increments the retried counter.
func (r *RDB) Retry(ctx context.Context, msg *base.TaskMessage, processAt time.Time, errMsg string, isFailure bool) error {
	var op errors.Op = "rdb.Retry"
	now := r.clock.Now()
	modified := *msg
	if isFailure {
		modified.Retried++
	}
	modified.ErrorMsg = errMsg
	modified.LastFailedAt = now.Unix()		//下一次触发的时间，属性关联于Redis的SortSet
	encoded, err := base.EncodeMessage(&modified)
	if err != nil {
		return errors.E(op, errors.Internal, fmt.Sprintf("cannot encode message: %v", err))
	}
	expireAt := now.Add(statsTTL)
	keys := []string{
		base.TaskKey(msg.Queue, msg.ID),
		base.ActiveKey(msg.Queue),
		base.LeaseKey(msg.Queue),
		base.RetryKey(msg.Queue),
		base.ProcessedKey(msg.Queue, now),
		base.FailedKey(msg.Queue, now),
		base.ProcessedTotalKey(msg.Queue),
		base.FailedTotalKey(msg.Queue),
	}
	argv := []interface{}{
		msg.ID,
		encoded,
		processAt.Unix(),
		expireAt.Unix(),
		isFailure,
		int64(math.MaxInt64),
	}
	return r.runScript(ctx, op, retryCmd, keys, argv...)
}

// KEYS[1] -> asynq:{<qname>}:t:<task_id>
// KEYS[2] -> asynq:{<qname>}:active
// KEYS[3] -> asynq:{<qname>}:lease
// KEYS[4] -> asynq:{<qname>}:retry
// KEYS[5] -> asynq:{<qname>}:processed:<yyyy-mm-dd>
// KEYS[6] -> asynq:{<qname>}:failed:<yyyy-mm-dd>
// KEYS[7] -> asynq:{<qname>}:processed
// KEYS[8] -> asynq:{<qname>}:failed
// -------
// ARGV[1] -> task ID
// ARGV[2] -> updated base.TaskMessage value
// ARGV[3] -> retry_at UNIX timestamp
// ARGV[4] -> stats expiration timestamp
// ARGV[5] -> is_failure (bool)
// ARGV[6] -> max int64 value
var retryCmd = redis.NewScript(`
if redis.call("LREM", KEYS[2], 0, ARGV[1]) == 0 then  
  return redis.error_reply("NOT FOUND")
end
if redis.call("ZREM", KEYS[3], ARGV[1]) == 0 then
  return redis.error_reply("NOT FOUND")
end
redis.call("ZADD", KEYS[4], ARGV[3], ARGV[1])
redis.call("HSET", KEYS[1], "msg", ARGV[2], "state", "retry")
if tonumber(ARGV[5]) == 1 then
	local n = redis.call("INCR", KEYS[5])
	if tonumber(n) == 1 then
		redis.call("EXPIREAT", KEYS[5], ARGV[4])
	end
	local m = redis.call("INCR", KEYS[6])
	if tonumber(m) == 1 then
		redis.call("EXPIREAT", KEYS[6], ARGV[4])
	end
    local total = redis.call("GET", KEYS[7])
    if tonumber(total) == tonumber(ARGV[6]) then
    	redis.call("SET", KEYS[7], 1)
    	redis.call("SET", KEYS[8], 1)
    else
    	redis.call("INCR", KEYS[7])
    	redis.call("INCR", KEYS[8])
    end
end
return redis.status_reply("OK")`)
```

## 0x02 metrics指标 

本小节主要看下asynq定义的[metrics](https://github.com/hibiken/asynq/tree/master/x/metrics)，一款异步队列中间件需要考虑哪些指标

#### 指标定义与说明 

```Go
// Descriptors used by QueueMetricsCollector
var (
	tasksQueuedDesc = prometheus.NewDesc(
		prometheus.BuildFQName(namespace, "", "tasks_enqueued_total"),
		"Number of tasks enqueued; broken down by queue and state.",
		[]string{"queue", "state"}, nil,
	)

	queueSizeDesc = prometheus.NewDesc(
		prometheus.BuildFQName(namespace, "", "queue_size"),
		"Number of tasks in a queue",
		[]string{"queue"}, nil,
	)

	queueLatencyDesc = prometheus.NewDesc(
		prometheus.BuildFQName(namespace, "", "queue_latency_seconds"),
		"Number of seconds the oldest pending task is waiting in pending state to be processed.",
		[]string{"queue"}, nil,
	)

	queueMemUsgDesc = prometheus.NewDesc(
		prometheus.BuildFQName(namespace, "", "queue_memory_usage_approx_bytes"),
		"Number of memory used by a given queue (approximated number by sampling).",
		[]string{"queue"}, nil,
	)

	tasksProcessedTotalDesc = prometheus.NewDesc(
		prometheus.BuildFQName(namespace, "", "tasks_processed_total"),
		"Number of tasks processed (both succeeded and failed); broken down by queue",
		[]string{"queue"}, nil,
	)

	tasksFailedTotalDesc = prometheus.NewDesc(
		prometheus.BuildFQName(namespace, "", "tasks_failed_total"),
		"Number of tasks failed; broken down by queue",
		[]string{"queue"}, nil,
	)

	pausedQueues = prometheus.NewDesc(
		prometheus.BuildFQName(namespace, "", "queue_paused_total"),
		"Number of queues paused",
		[]string{"queue"}, nil,
	)
)
```

## 0x03 总结 

本文介绍了asynq的实现，asynq的代码非常值得一读，实现思路非常清晰


# 爆破 💥 Asynq


我在 Golang 的生态里一直没找到好用的延时任务方案，在之前的工作中需要的时候也是自己[实现了一个简易方案](https://www.jianshu.com/p/83f37db7b078)，它有诸多问题，比如部署复杂，稳定性不够。

直到 2022 年我知晓了 [Asynq](https://github.com/hibiken/asynq)。

Asynq 这样介绍自己：

> Simple, reliable & efficient distributed task queue in Go
> 
> Go 中简单、可靠、高效的分布式任务队列
> 
> Asynq is a Go library for queueing tasks and processing them asynchronously with workers. It's backed by Redis and is designed to be scalable yet easy to get started.
> 
> Asynq 是一个 Go 库，用于排队任务并通过工作者异步处理这些任务。它由 Redis 支持，被设计成可扩展且容易上手。

Asynq 过去一年在公司业务中调度了超过 5000w 个任务，确实高效且稳定，并且使用的 Redis 人手都有，部署难度很低。

现在空下来研究下它的实现原理，发现比想象中更复杂也有趣。

## 在此之前的思考

在研究 Asynq 之前，我也尝试自己思考了下应该如何实现，这样才知道有哪些难点，以及 Asyncq 如此设计是为了解决什么问题。

实现方案：使用有序集合存储将要到期的任务，定时从集合中拿取任务，看起来我们只需要用到一个数据结构，一个 key 就可以了。

问题：

1. 如果有多个节点同时执行，那么这些节点可能会拿到同一个过期的任务，导致重复执行。
2. 如果任务执行失败或者无响应，如何重试？

解决问题 1 可以先拿出再删除，如果没删除成功，说明已经有其他节点拿到了这个任务，只有删除成功才能执行。

问题 2 更复杂，程序逻辑无法解决这个问题，因为程序本身就不稳定， 所以仅仅使用问题 1 的解决方案是实现不了的，因为删除后数据就从 Redis 丢失了，而程序又无法可靠的进行重试。

我们还需要一个新的 key 存储等待响应的任务，在任务超时后重试。现在问题变得复杂起来了，放弃思考，抄答案吧。

带着这两个问题，我们来看看 Asyncq 它是如何实现的。

## Asynq 总体流程

### 入队

- 使用 有序集合 `scheduled` 记录异步任务。
- 如果不是延时任务（立即执行），则直接加入到 `pending` 队列。

### 前进

- 使用 `ZRANGEBYSCORE` 定时从 `scheduled` 和 `retry` 队列查询到期的任务，并使用 `ZREM` 删除 任务，并插入到 `pending` 队列。 （！必须使用 script 才能满足原子性）

### 出队

- 使用 `RPOPLPUSH` 将`pending` 的任务放入`active` 队列，并将任务放在有序集合 `lease`（延时 30 s）

### 执行任务

- 成功执行了之后 将任务从 `active` 和 `lease` 移除，然后存入归档。
- 执行失败，执行 retry：将任务从 `active` 和 `lease` 移除，（如果没有移除成功，则不做任何处理），放入有序集合 `retry` （延时 N s）
- 如果服务正在退出则重入队列：将任务从 `active` 移除，从`lease` 移除，添加到 `pending` 队列。
- 超时没有 ACK：定时从 `lease` 取出任务，检查最大重试次数，如果没有超过最大重试次数就执行 retry。否则就存档标记为失败。（注意这里可以不使用 script 实现原子操作，因为 retry 命令是幂等的。）

> 流程图可能画得不够好，将就看看辅助理解。

## 要点

### 保证原子性

> 原子性是指操作在执行过程中不可被中断或分割，要么全部执行成功，要么全部失败回滚，保证操作的完整性和一致性。 在并发环境下，原子性是确保多个线程在同时访问共享资源时，对共享资源的操作不会相互干扰从而导致数据不一致。

由于在某个动作中需要操作多个队列，为了实现原子性，则需要使用 script 命令来执行操作，比如一次出队列操作如下：

```go
var dequeueCmd = redis.NewScript(`
if redis.call("EXISTS", KEYS[2]) == 0 then
	local id = redis.call("RPOPLPUSH", KEYS[1], KEYS[3])
	if id then
		local key = ARGV[2] .. id
		redis.call("HSET", key, "state", "active")
		redis.call("HDEL", key, "pending_since")
		redis.call("ZADD", KEYS[4], ARGV[1], id)
		return redis.call("HGET", key, "msg")
	end
end
return nil`)


```

### 保证至少执行一次

由于程序的崩溃可能发生在任何地方，为了保证不丢数据，和 MQ 一样，我们必须设计一个 ACK（确认 Acknowledgement）机制，如果超过一段时间程序没有应答（无论成功和失败都算应答）则需要重试。这也是整个系统最复杂的地方，可以说将复杂度扩大了几倍。

在 asyncq 中出队列时，不光使用 `RPOPLPUSH` 将 `pending` 的任务放入 `active` 队列，同时也会将任务放在有序集合 `lease`（默认延时 30 s），然后定时从 `lease` 查询出过期的任务并重试。

### 避免重复执行

并发可能导致判断失效而重复执行，asynq 在不同场景使用不同机制来避免重复执行：

- 出队列：asyncq 中使用了 redis script 实现并发控制（先查询后删除两步操作）来避免一个任务被多次查询出来并执行。
- 超时重试：1. 定时从 `lease` 查询出过期的任务，2. 再检查重试次数之后重新放入 `retry` 集合，并从 `active` 和 `lease` 移除任务， 由于步骤 1 和 2 无法像出队列时做在一个 script 中，所以当多个 pod 并发查询时会查询到同一个任务，如果都往 `retry` 集合放入的话就会导致重复执行 为了避免这个问题，可以先从 `active` 和 `lease` 移除任务，只有当移除成功之后才会放入 `retry` 集合。

# Reference
https://pandaychen.github.io/2021/08/18/A-GOLANG-ASYNQ-ANALYSIS/
https://blog.bysir.top/blogs/boom_asynq/

