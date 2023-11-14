---
title: ✨ redis-延迟队列
date created: 2023-11-11
date modified: 2023-11-13
---
> [!TIP] 基于redis实现延迟队列💡  
>  首先，先界定讨论的边界。这里基于redis的延迟队列。可以用于
>  1. 为不支持延迟队列的mq，进行开发延迟队列功能的二开方案。相当于消费者是重投到mq中。如果是重投服务的延迟队列，这里应要注意支持扩容，各个timer的负载，容错问题。当前仅仅讨论的是存储层的设计
>  2. 消费者直接进行消费，如直接使用redission  
>  不建议直接使用redis进行消费。具体原因，可以看下[[redis-消息队列#^84080d]]，redis与专业的消息队列的区别。可参考[[rocketmq-延迟队列]]

## 思路，参考[千万级延时任务队列如何实现](https://zhuanlan.zhihu.com/p/94082947)

[processon](https://www.processon.com/diagraming/6551d0fa28841565d1d1be54)  
![延迟队列.jpg](http://image.clickear.top/%E5%BB%B6%E8%BF%9F%E9%98%9F%E5%88%97.jpg)

## 核心功能介绍

这里，介绍整体的延迟队列的实现。

### 名词介绍

Producer: 生产者，用于发送延迟队列消息。  
Consumer: 消费者，用于处理业务逻辑。  
Timer: 定时器服务，主要将到达执行时间的消息，转移到ReadyQueue队列中。ps： 这里要注意使用原子操作，可以使用lua脚本进行操作。  
ReadyQueue: 已经到达执行时间的queue。消费者可以直接进行消费。  
DeadQueue: 消费失败的队列。  
PoolKV： 消息的kv键值对数据。单独存放，主要是为了方便[[序列化器 |序列化]]和[[压缩算法 |压缩]]。一般消息体都会比较大。

### 整体流程

#### 生产者发送消息

1.0 将消息放到 pool中，将消息体单独剥离出来，方便消息体进行序列化和压缩。  
1.1 如果延迟消息已经到达执行时间，则直接放入到消息队列中  
1.2 如果延迟消息未到达执行时间，则将消息放入到zset中，以执行时间为sort。value为 msgId

#### 到期消息，转移到队列

Timer服务端，查询到期消息，放入到ReadyQueue中  

问题:  
zset如何避免大key?

1. 可通过按小时维度，进行拆分key

获取到期消息方案:

1. 一直轮询zset，不建议使用。浪费性能
2. 通过[[时间轮]]，将快过期的消息，放到时间轮中进行消费。
	1. 放到时间轮的数据，有哪些？全量数据还是部分？
		1. 放到时间轮的数据，肯定不能全量数据，应该是最近快要到期的数据.即延迟加载。要注意的时候，如果按照时间进行分割，边界时间可能会遗漏的情况，所以需要加载临近的2个时间轮数据
	2. 什么时候放到时间轮中？

#### 消费者消费

3.1 消费者，消费readyQueue，如果消费失败，可以将消息放到 DeadQueue中

### 注意问题

#### zset的大key以及热点key问题

1. 按照时间维度进行拆分
2. 使用 [[序列化器 |序列化]]和[[压缩算法 |压缩]]

#### 时间轮的数据量问题

[延迟加载](https://www.cnblogs.com/hzmark/p/mq-delay-msg.html)，不是全量数据，而是加载最近要调度的数据，或者与redission一样，仅加载最新的head的task。

#### 如果按时间拆分zset，需要跨时间加载到时间轮的边界问题

加载临近时间区域的数据

## 开源实现分析

|            | 存储数据的结构                                            | 获取zset数据方式                           | 共有实现 |     |
|:---------- |:--------------------------------------------------------- |:------------------------------------------ |:-------- |:--- |
| redission  | zset + list  + list(存储消息) + channel(通知最新消息变更) | 使用channel,只有最新一个task，放入时间轮中 | 使用lua，保证原子性         |     |
| 美图lmstfy | zset+ list + string(存储消息)                             | 定时轮询zset数据                           |  使用lua，保证原子性        |     |

### redission的[DelayedQueue](https://www.nowcoder.com/discuss/459290197163823104) java

![image.png](http://image.clickear.top/20231113164843.png)

1. redission_delay_queue_timeout ==> zset。score 是元素的过期时间，按从小到大排序。如果到达过期时间，会转移到 list中
2. redission_delay_queue: 消息体集合，按插入顺序排序。类比与 pool kv,但是这里是使用list进行存储。
3. redission_queue_channel ==> channel，利用channel进行触发获取转移任务功能。即消息的入[[时间轮]]操作。
	1. 有新的订阅者，尝试获取zset数据，或者最新的任务的时间。
	2. 监听message，如果有新的message，将数据push到时间轮中。
4. readyQueue,到达执行时间后，将zset的消息，转移到readyQueue中。消费者只监听该队列即可。

存在问题:

1. zset存在大key的风险。
2. consume消费消息，使用的是blpop.有可能消费者消费失败，丢失数据的风险。

#### 生产者

#### 使用

```java
 public <T> void add(T t, long delay, TimeUnit timeUnit, String queueName) {  
        // readyQueue
        RBlockingQueue<T> blockingFairQueue = redissonClient.getBlockingQueue(queueName);  
        RDelayedQueue<T> delayedQueue = redissonClient.getDelayedQueue(blockingFairQueue);  
        // 添加消息
        delayedQueue.offer(t, delay, timeUnit);  
    }
```

#### RedissonDelayedQueue初始化

```java
public class RedissonDelayedQueue<V> extends RedissonExpirable implements RDelayedQueue<V> {  
  
    private final QueueTransferService queueTransferService;  
    private final String channelName;  
    private final String queueName;  
    private final String timeoutSetName;  
      
    protected RedissonDelayedQueue(QueueTransferService queueTransferService, Codec codec, final CommandAsyncExecutor commandExecutor, String name) {  
        super(codec, commandExecutor, name);  
        // 订阅通道名前缀  
        channelName = prefixName("redisson_delay_queue_channel", getName());  
        // 延时队列 List 前缀  
        queueName = prefixName("redisson_delay_queue", getName());  
        // 延时队列过期排序 Sorted Set 的 KEY 前缀  
        timeoutSetName = prefixName("redisson_delay_queue_timeout", getName());  
        // 创建一个订阅作为监听器  
        QueueTransferTask task = new QueueTransferTask();
        // 执行任务  
        queueTransferService.schedule(queueName, task);  
          
        this.queueTransferService = queueTransferService;  
    }  
  
    @Override  
    public void offer(V e, long delay, TimeUnit timeUnit) {  
        // 添加元素  
        get(offerAsync(e, delay, timeUnit));  
    }  
      
    @Override  
    public RFuture<Void> offerAsync(V e, long delay, TimeUnit timeUnit){ 
	    // ... 详见发送消息
    }  
}
```

1. 初始化，监听channel任务

```java
// org.redisson.QueueTransferTask#start
public void start() {  
    RTopic schedulerTopic = getTopic();  
    statusListenerId = schedulerTopic.addListener(new BaseStatusListener() {  
        @Override  
        public void onSubscribe(String channel) {  
            // 尝试进行转移zset数据，并获取最新的待执行任务
            pushTask();  
        }  
    });  
    messageListenerId = schedulerTopic.addListener(Long.class, new MessageListener<Long>() {  
        @Override  
        public void onMessage(CharSequence channel, Long startTime) { 
            // 监听消息，如果channel有消息，则放入到时间轮中。 
            scheduleTask(startTime);  
        }  
    });  
}
```

#### 发送消息

步骤:

1. 插入到zset中， zadd
2. 插入到消息列表中， rpush
3. 如果最新的消息，是head任务，则通知所有worker进行更新时间轮。(通过发送channel中)

```java

   public RFuture<Void> offerAsync(V e, long delay, TimeUnit timeUnit) {  
        if (delay < 0) {  
            throw new IllegalArgumentException("Delay can't be negative");  
        }  
        // 延时时间转换为毫秒值  
        long delayInMs = timeUnit.toMillis(delay);  
        // 超时时间=当前时间毫秒值 + 延时时间毫秒值  
        long timeout = System.currentTimeMillis() + delayInMs;  
       
        long randomId = ThreadLocalRandom.current().nextLong(); 
        // 执行lua脚本，插入到zset中 
        return commandExecutor.evalWriteAsync(getName(), codec, RedisCommands.EVAL_VOID,  
                "local value = struct.pack('dLc0', tonumber(ARGV[2]), string.len(ARGV[3]), ARGV[3]);"   
              + "redis.call('zadd', KEYS[2], ARGV[1], value);"  
              + "redis.call('rpush', KEYS[3], value);"  
              // if new object added to queue head when publish its startTime   
              // to all scheduler workers   
              + "local v = redis.call('zrange', KEYS[2], 0, 0); "  
              + "if v[1] == value then "  
                 + "redis.call('publish', KEYS[4], ARGV[1]); "  
              + "end;",  
              Arrays.<Object>asList(getName(), timeoutSetName, queueName, channelName),   
              timeout, randomId, encode(e));  
    } 
```

#### 消息转移服务

1. 监听channel,将最新的head任务，放到时间轮中。  
将可执行的消息，移除zset和list，

1. 通过监听channel，
	1. 如果有新的订阅者，则尝试进行推送任务
	2. 监听消息，如果有最新的待执行head任务，则会发送到channel。更新时间轮的数据
2. pushTask
3. pushTaskAsync，返回最新的head任务的时间戳。
	1. 获取过期的延迟消息数据。push到readyQueue，并放到readyQueue中
	2. 最新的head任务的时间戳
	3. 执行scheduleTask。
4. scheduleTask，更新时间轮或者直接再次执行pushTask

```java
public abstract class QueueTransferTask {
    
    /**
     ** 初始化
     **/
    public void start() {
        RTopic schedulerTopic = getTopic();
        statusListenerId = schedulerTopic.addListener(new BaseStatusListener() {
            @Override
            public void onSubscribe(String channel) {
                pushTask();
            }
        });
        
        messageListenerId = schedulerTopic.addListener(Long.class, new MessageListener<Long>() {
            @Override
            public void onMessage(CharSequence channel, Long startTime) {
                scheduleTask(startTime);
            }
        });
    }
    
   
    private void scheduleTask(final Long startTime) {
        // 只保存最新的待执行的时间戳，入队列中
        TimeoutTask oldTimeout = lastTimeout.get();
        if (startTime == null) {
            return;
        }
        
        if (oldTimeout != null) {
            oldTimeout.getTask().cancel();
        }
        
        long delay = startTime - System.currentTimeMillis();
        if (delay > 10) {
            Timeout timeout = connectionManager.newTimeout(new TimerTask() {                    
                @Override
                public void run(Timeout timeout) throws Exception {
                    pushTask();
                    
                    TimeoutTask currentTimeout = lastTimeout.get();
                    if (currentTimeout.getTask() == timeout) {
                        lastTimeout.compareAndSet(currentTimeout, null);
                    }
                }
            }, delay, TimeUnit.MILLISECONDS);
            if (!lastTimeout.compareAndSet(oldTimeout, new TimeoutTask(startTime, timeout))) {
                timeout.cancel();
            }
        } else {
            pushTask();
        }
    }
    
    protected abstract RTopic getTopic();
    
    protected RFuture<Long> pushTaskAsync(){
        return commandExecutor.evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_LONG,
                        "local expiredValues = redis.call('zrangebyscore', KEYS[2], 0, ARGV[1], 'limit', 0, ARGV[2]); "
                      + "if #expiredValues > 0 then "
                          + "for i, v in ipairs(expiredValues) do "
                              + "local randomId, value = struct.unpack('dLc0', v);"
                              + "redis.call('rpush', KEYS[1], value);"
                              + "redis.call('lrem', KEYS[3], 1, v);"
                          + "end; "
                          + "redis.call('zrem', KEYS[2], unpack(expiredValues));"
                      + "end; "
                        // get startTime from scheduler queue head task
                      + "local v = redis.call('zrange', KEYS[2], 0, 0, 'WITHSCORES'); "
                      + "if v[1] ~= nil then "
                         + "return v[2]; "
                      + "end "
                      + "return nil;",
                      Arrays.<Object>asList(getRawName(), timeoutSetName, queueName),
                      System.currentTimeMillis(), 100);
    }
    
    private void pushTask() {
        RFuture<Long> startTimeFuture = pushTaskAsync();
        startTimeFuture.onComplete((res, e) -> {
            if (e != null) {
                if (e instanceof RedissonShutdownException) {
                    return;
                }
                log.error(e.getMessage(), e);
                scheduleTask(System.currentTimeMillis() + 5 * 1000L);
                return;
            }
            
            if (res != null) {
                scheduleTask(res);
            }
        });
    }
}
```

#### 消费者

```java
 private final RDelayedQueue<String> delayedQueue;
    private final RBlockingQueue<String> blockingQueue;
    @PostConstruct
    public void init() {
        ExecutorService executorService = Executors.newFixedThreadPool(1);
        executorService.submit(() -> {
            while (true) {
                try {
                    String task = blockingQueue.take();
                    log.info("rev delay task:{}", task);
                } catch (Exception e) {
                    log.error("occur error", e);
                }
            }
        });
    }
```

### 美图 [lsmtfy](https://github.com/bitleak/lmstfy) go实现

#### 生产者

``` go
    // 消息体，放到pool中
	err = e.pool.Add(job)
	if err != nil {
		return job.ID(), fmt.Errorf("pool: %s", err)
	}
    // 已经到期的，直接执行，放入到queue中
	if delaySecond == 0 {
		q := NewQueue(namespace, queue, e.redis, e.timer)
		err = q.Push(job, tries)
		if err != nil {
			err = fmt.Errorf("queue: %s", err)
		}
		return job.ID(), err
	}
	// 未到期的，放入到zset中
	err = e.timer.Add(namespace, queue, job.ID(), delaySecond, tries)
	if err != nil {
		err = fmt.Errorf("timer: %s", err)
	}}
```

pool.Add 直接使用setnx，进行存储

```go 
// e.pool.Add(job)
func (p *Pool) Add(j engine.Job) error {
	body := j.Body()
	metrics.poolAddJobs.WithLabelValues(p.redis.Name).Inc()
	// SetNX return OK(true) if key didn't exist before.
	ok, err := p.redis.Conn.SetNX(dummyCtx, PoolJobKey(j), body, time.Duration(j.TTL())*time.Second).Result()
	if err != nil {
		// Just retry once.
		ok, err = p.redis.Conn.SetNX(dummyCtx, PoolJobKey(j), body, time.Duration(j.TTL())*time.Second).Result()
	}
	if err != nil {
		return err
	}
	if !ok {
		return errors.New("key existed") // Key existed before, avoid overwriting it, so return error
	}
	return err
}
```

timer.Add

```go
func (t *Timer) Add(namespace, queue, jobID string, delaySecond uint32, tries uint16) error {
	metrics.timerAddJobs.WithLabelValues(t.redis.Name).Inc()
	timestamp := time.Now().Unix() + int64(delaySecond)

	// struct-pack the data in the format `Hc0Hc0HHc0`:
	//   {namespace len}{namespace}{queue len}{queue}{tries}{jobID len}{jobID}
	// length are 2-byte uint16 in little-endian
	namespaceLen := len(namespace)
	queueLen := len(queue)
	jobIDLen := len(jobID)
	buf := make([]byte, 2+namespaceLen+2+queueLen+2+2+jobIDLen)
	binary.LittleEndian.PutUint16(buf[0:], uint16(namespaceLen))
	copy(buf[2:], namespace)
	binary.LittleEndian.PutUint16(buf[2+namespaceLen:], uint16(queueLen))
	copy(buf[2+namespaceLen+2:], queue)
	binary.LittleEndian.PutUint16(buf[2+namespaceLen+2+queueLen:], uint16(tries))
	binary.LittleEndian.PutUint16(buf[2+namespaceLen+2+queueLen+2:], uint16(jobIDLen))
	copy(buf[2+namespaceLen+2+queueLen+2+2:], jobID)

	return t.redis.Conn.ZAdd(dummyCtx, t.Name(), &redis.Z{Score: float64(timestamp), Member: buf}).Err()
}
```

轮询zset

```go
const (
	luaPumpQueueScript = `
local zset_key = KEYS[1]
local output_queue_prefix = KEYS[2]
local pool_prefix = KEYS[3]
local output_deadletter_prefix = KEYS[4]
local now = ARGV[1]
local limit = ARGV[2]

local expiredMembers = redis.call("ZRANGEBYSCORE", zset_key, 0, now, "LIMIT", 0, limit)

if #expiredMembers == 0 then
	return 0
end

for _,v in ipairs(expiredMembers) do
	local ns, q, tries, job_id = struct.unpack("Hc0Hc0HHc0", v)
	if redis.call("EXISTS", table.concat({pool_prefix, ns, q, job_id}, "/")) > 0 then
		-- only pump job to ready queue/dead letter if the job did not expire
		if tries == 0 then
			-- no more tries, move to dead letter
			local val = struct.pack("HHc0", 1, #job_id, job_id)
			redis.call("PERSIST", table.concat({pool_prefix, ns, q, job_id}, "/"))  -- remove ttl
			redis.call("LPUSH", table.concat({output_deadletter_prefix, ns, q}, "/"), val)
		else
			-- move to ready queue
			local val = struct.pack("HHc0", tonumber(tries), #job_id, job_id)
			redis.call("LPUSH", table.concat({output_queue_prefix, ns, q}, "/"), val)
		end
	end
end
redis.call("ZREM", zset_key, unpack(expiredMembers))
return #expiredMembers
`
)
func (t *Timer) tick() {
	tick := time.NewTicker(t.interval)
	for {
		select {
		case now := <-tick.C:
			currentSecond := now.Unix()
			t.pump(currentSecond)
		case <-t.shutdown:
			return
		}
	}
}

func (t *Timer) pump(currentSecond int64) {
	for {
		val, err := t.redis.Conn.EvalSha(dummyCtx, t.pumpSHA, []string{t.Name(), QueuePrefix, PoolPrefix, DeadLetterPrefix}, currentSecond, BatchSize).Result()
		if err != nil {
			if isLuaScriptGone(err) { // when redis restart, the script needs to be uploaded again
				sha, err := t.redis.Conn.ScriptLoad(dummyCtx, luaPumpQueueScript).Result()
				if err != nil {
					logger.WithField("err", err).Error("Failed to reload script")
					time.Sleep(time.Second)
					return
				}
				t.pumpSHA = sha
			}
			logger.WithField("err", err).Error("Failed to pump")
			time.Sleep(time.Second)
			return
		}
		n, _ := val.(int64)
		logger.WithField("count", n).Debug("Due jobs")
		metrics.timerDueJobs.WithLabelValues(t.redis.Name).Add(float64(n))
		if n == BatchSize {
			// There might have more expired jobs to pump
			metrics.timerFullBatches.WithLabelValues(t.redis.Name).Inc()
			time.Sleep(10 * time.Millisecond) // Hurry up! accelerate pumping the due jobs
			continue
		}
		return
	}
}
```

## 资料

[一种优雅的redis延迟队列实现\_牛客网](https://www.nowcoder.com/discuss/459290197163823104)  
[延时队列 这 Redisson DelayedQueue 实现 - 光星の博客](http://blog.gxitsky.com/2021/03/15/ArchitectureDesign-Distrbuted-Task-DelayQueue-Redisson/)  
[美图开源的lsmtfy,千万级延时任务队列如何实现](https://zhuanlan.zhihu.com/p/94082947)
