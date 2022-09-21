---
date created: 2022-09-19
date modified: 2022-09-19
title: ∑ mq
---

## 关键词

异步、削峰、解耦、消息分发、大事务拆分成 `(小事务+ 异步)`、错误重试、消费者幂、分布式、**重复消费、消息丢失、顺序消费**等

## 什么是消息队列(MQ)?

> 队列是一种先进先出的数据结构。但是作为中间件，跟单机中的队列是不一样的。 毫无疑问，单机的队列是无法满足需求的，集群/分布式。 消除了单点故障、保证消息可靠性等。消息队列可以简单理解为：把要传输的数据放在队列中。消息队列做业务解耦/最终一致性/广播/错峰流控等。反之，如果需要强一致性，关注业务逻辑的处理结果，则RPC显得更为合适。

### 名称定义

- 消费者:
- 生产者
- broker

### 优点

#### 解耦

> 作为一个订单系统，我们可以需要在创建订单、有库存信息或者有物流信息调用其它子系统。耦合严重。如某个系统挂了，导致该接口异常。将消息发送到队列中，需要使用的系统，去订阅的方式  
![](http://image.clickear.top/20220919091014.png)

#### 异步

> 非必要的操作，异步去处理。而不是同步。比如下单支付完之后，发送等短信，发送短信的结果不应该影响 用户下单，这样异步化，可以提供接口qps。当然，异步的方式有很多，比如你开启线程池处理等。  
![](http://image.clickear.top/20220919091047.png)

#### 削峰

> 在秒杀或者有活动的时候，通过先把请求放到队列里面，然后至于每秒消费多少请求，就看自己的服务器处理能力，你能处理5000QPS你就消费这么多，可能会比正常的慢一点，但是不至于打挂服务器。等活动结束之后，可以慢慢消费队列中的数据。比如3更半夜去消费等。比如常见的压单等操作

### 缺点

#### 数据一致性

> 下单的服务自己保证自己的逻辑成功处理了，你成功发了消息，但是优惠券系统，积分系统等等这么多系统，他们成功还是失败你就不管了？这里就涉及到[[分布式事务]]、[[最终一致性]]等问题。

#### 可用性

> 复杂度增加，要考虑 分布式服务的可用性。

#### 系统复杂度提高

> 硬生生加个 MQ 进来，你怎么[保证消息没有重复消费](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-that-messages-are-not-repeatedly-consumed.md)？怎么[处理消息丢失的情况](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-the-reliable-transmission-of-messages.md)？怎么保证消息传递的顺序性？头大头大，问题一大堆，痛苦不已。

## 适用场景

> 消息队列做业务解耦/最终一致性/广播/错峰流控等。反之，如果需要强一致性，关注业务逻辑的处理结果，则RPC显得更为合适。

### 上游不关心下游的执行结果

生产者不需要从消费者处获得反馈。如同步数据、发送通知。

#### 容许短暂的不一致性

## [技术选型对比](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/why-mq.md)

![](http://image.clickear.top/20220919091854.png)  
综上，各种对比之后，有如下建议：

一般的业务系统要引入 MQ，最早大家都用 ActiveMQ，但是现在确实大家用的不多了，没经过大规模吞吐量场景的验证，社区也不是很活跃，所以大家还是算了吧，我个人不推荐用这个了；

后来大家开始用 RabbitMQ，但是确实 erlang 语言阻止了大量的 Java 工程师去深入研究和掌控它，对公司而言，几乎处于不可控的状态，但是确实人家是开源的，比较稳定的支持，活跃度也高；

不过现在确实越来越多的公司会去用 RocketMQ，确实很不错，毕竟是阿里出品，但社区可能有突然黄掉的风险（目前 RocketMQ 已捐给 [Apache](https://github.com/apache/rocketmq)，但 GitHub 上的活跃度其实不算高）对自己公司技术实力有绝对自信的，推荐用 RocketMQ，否则回去老老实实用 RabbitMQ 吧，人家有活跃的开源社区，绝对不会黄。

所以**中小型公司**，技术实力较为一般，技术挑战不是特别高，用 RabbitMQ 是不错的选择；**大型公司**，基础架构研发实力较强，用 RocketMQ 是很好的选择。

如果是**大数据领域**的实时计算、日志采集等场景，用 Kafka 是业内标准的，绝对没问题，社区活跃度很高，绝对不会黄，何况几乎是全世界这个领域的事实性规范。

### RocketMQ与Kafka对比

#### 数据可靠性

- RocketMQ支持异步实时刷盘，同步刷盘，同步Replication，异步Replication
- Kafka使用异步刷盘方式，异步Replication/同步Replication

RocketMQ的同步刷盘在单机可靠性上比Kafka更高，不会因为操作系统Crash，导致数据丢失。

Kafka同步Replication理论上性能低于RocketMQ的同步Replication，原因是Kafka的数据以分区为单位组织，意味着一个Kafka实例上会有几百个数据分区，RocketMQ一个实例上只有一个数据分区，RocketMQ可以充分利用IO Group Commit机制，批量传输数据，配置同步Replication与异步Replication相比，性能损耗约20%~30%

#### 性能对比

- Kafka单机写入TPS约在百万条/秒，消息大小10个字节
- RocketMQ单机写入TPS单实例约7万条/秒，单机部署3个Broker，可以跑到最高12万条/秒，消息大小10个字节

Kafka的TPS跑到单机百万，主要是由于Producer端将多个小消息合并，批量发向Broker。

RocketMQ为什么没有这么做？

1. Producer通常使用Java语言，缓存过多消息，GC是个很严重的问题
2. Producer调用发送消息接口，消息未发送到Broker，向业务返回成功，此时Producer宕机，会导致大量消息丢失，业务出错
3. Producer通常为分布式系统，且每台机器都是多线程发送，我们认为线上的系统单个Producer每秒产生的数据量有限，不可能上万。
4. 缓存的功能完全可以由上层业务完成。

#### 单机支持的队列数

- Kafka单机超过64个队列/分区，Load会发生明显的飙高现象，队列越多，load越高，发送消息响应时间变长。
- RocketMQ单机支持最高5万个队列，Load不会发生明显变化

队列多有什么好处？

单机可以创建更多Topic，因为每个Topic都是由一批队列组成

Consumer的集群规模和队列数成正比，队列越多，Consumer集群可以越大

#### 严格的消息顺序

- Kafka支持消息顺序，但是一台Broker宕机后，就会产生消息乱序
- RocketMQ支持严格的消息顺序，在顺序消息场景下，一台Broker宕机后，发送消息会失败，但是不会乱序

#### 分布式事务消息

- Kafka支持分布式事务消息
- [[RocketMQ事务消息]]是支持的  
但是两者对事务的区分度不一样。

+ RocketMQ 解决的是**本地事务的执行和发消息这两个动作满足事务**的约束。
+ Kafka 事务消息则是用**在一次事务中需要发送多个消息**的情况，保证多个消息之间的事务约束，即多条消息要么都发送成功，要么都发送失败。

#### 消息查询

- <del>Kafka不支持消息查询</del> 现在应该是支持的。具体未调研
- RocketMQ支持根据Message Id查询消息，也支持根据消息内容查询消息（发送消息时指定一个Message Key，任意字符串，例如指定为订单Id）

#### 高性能的存储

- Kafka在Topic数量由64增长到256时，吞吐量下降了98.37%。
- RocketMQ在Topic数量由64增长到256时，吞吐量只下降了16%。  
1. kafka一个topic下面的所有消息都是以partition的方式分布式的存储在多个节点上。同时在kafka的机器上，**每个Partition**其实都会对应一个日志目录，在目录下面会对应多个日志分段。所以如果Topic很多的时候Kafka虽然写文件是顺序写，但实际上文件过多，会造成**磁盘IO竞争**非常激烈。
2. rocketmq消息主体数据并没有像Kafka一样写入多个文件，而是**写入一个文件**,这样我们的写入IO竞争就非常小，可以在很多Topic的时候依然保持很高的吞吐量。虽然ConsumeQueue写是在不停的写入，并且ConsumeQueue是以Queue维度来创建文件，文件数量依然很多，但是**ConsumeQueue**的**写入的数据量很小**，每条消息只有20个字节，30W条数据也才6M左右，所以其实对我们的影响相对Kafka的Topic之间影响是要小很多的

#### 读取消息

Kafka中每个Partition都会是一个单独的文件，所以当消费某个消息的时候，会很好的出现顺序读，我们知道OS从物理磁盘上访问读取文件的同时，会顺序对其他相邻块的数据文件进行预读取，将数据放入PageCache，所以Kafka的读取消息性能比较好。

RocketMQ读取流程如下：

- 先读取ConsumerQueue中的offset对应CommitLog物理的offset
- 根据offset读取CommitLog

ConsumerQueue也是每个Queue一个单独的文件，并且其文件体积小，所以很容易利用PageCache提高性能。而CommitLog，由于同一个Queue的连续消息在CommitLog其实是不连续的，所以会造成随机读，RocketMQ对此做了几个优化：

- Mmap映射读取，Mmap的方式减少了传统IO将磁盘文件数据在操作系统内核地址空间的缓冲区和用户应用程序地址空间的缓冲区之间来回进行拷贝的性能开销
- 使用DeadLine调度算法+SSD存储盘
- 由于Mmap映射受到内存限制，当不在Mmmap映射这部分数据的时候(也就是消息堆积过多)，默认是内存的40%，会将请求发送到SLAVE,减缓Master的压力

### 为什么kafaka的topic，吞吐量会下降？

[你应该知道的RocketMQ – ITPUB](http://www.itpub.net/2019/11/27/4449/)

1. kafka一个topic下面的所有消息都是以partition的方式分布式的存储在多个节点上。同时在kafka的机器上，**每个Partition**其实都会对应一个日志目录，在目录下面会对应多个日志分段。所以如果Topic很多的时候Kafka虽然写文件是顺序写，但实际上文件过多，会造成**磁盘IO竞争**非常激烈。
2. rocketmq消息主体数据并没有像Kafka一样写入多个文件，而是**写入一个文件**,这样我们的写入IO竞争就非常小，可以在很多Topic的时候依然保持很高的吞吐量。虽然ConsumeQueue写是在不停的写入，并且ConsumeQueue是以Queue维度来创建文件，文件数量依然很多，但是**ConsumeQueue**的**写入的数据量很小**，每条消息只有20个字节，30W条数据也才6M左右，所以其实对我们的影响相对Kafka的Topic之间影响是要小很多的

## mq的高可用

### rocketmq

一个 Topic 分布在多个Broker 上，一个 Broker 可以配置多个 Topic，它们是多对多的关系。如果某个 Topic 消息量很大，应该给它多配置几个队列，并且尽量多分布在不同 Broker 上，以减轻某个 Broker 的压力。Topic 消息量都比较均匀的情况下，如果某个 broker 上的队列越多，则该 broker 压力越大。[架构(Architecture) · Apache RocketMQ开发者指南](https://www.itmuch.com/books/rocketmq/architecture.html)

*写数据*的时候，生产者就写 leader，超过半数才算成功。 基于raft协议

### kafaka

[消息队列高可用](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-high-availability-of-message-queues.md)

Kafka 一个最基本的架构认识：由多个 broker 组成，每个 broker 是一个节点；你创建一个 topic，这个 topic 可以划分为多个 partition，每个 partition 可以存在于不同的 broker 上，每个 partition 就放一部分数据。  
这就是**天然的分布式消息队列**，就是说一个 topic 的数据，是**分散放在多个机器上的，每个机器就放一部分数据**  
**写数据**的时候，生产者就写 leader，然后 leader 将数据落地写本地磁盘，接着其他 follower 自己主动从 leader 来 pull 数据。一旦所有 follower 同步好数据了，就会发送 ack 给 leader，leader 收到所有 follower 的 ack 之后，就会返回写成功的消息给生产者。  
在 Kafka 服务端设置 `min.insync.replicas` 参数：这个值必须大于 1，这个是要求一个 leader 至少感知到有至少一个 follower 还跟自己保持联系，没掉队，这样才能确保 leader 挂了还有一个 follower 吧。  
**消费**的时候，只会从 leader 去读，但是只有当一个消息已经被所有 follower 都同步成功返回 ack 的时候，这个消息才会被消费者读到。

## 消息丢失

怎么[处理消息丢失的情况](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-the-reliable-transmission-of-messages.md)？

- **消息生产阶段：** 从消息被生产出来，然后提交给 MQ 的过程中，只要能正常收到 MQ Broker 的 ack 确认响应，就表示发送成功，所以只要处理好返回值和异常，这个阶段是不会出现消息丢失的。
- **消息存储阶段：** 这个阶段一般会直接交给 MQ 消息中间件来保证，但是你要了解它的原理，比如 Broker 会做副本，保证一条消息至少同步两个节点再返回 ack。 **开启持久化、集群 + 数据副本、刷盘机制**
- **消息消费阶段：** 消费端从 Broker 上拉取消息，只要消费端在收到消息后，不立即发送消费确认给 Broker，而是等到执行完业务逻辑后，再发送消费确认，也能保证消息的不丢失。**ACK确认机制**
- 降级补偿机制: 生产者持久化 + 消费者处理回调保障

|        | rocketMq                                     | kafaka | rabbitmq |
| ------ | -------------------------------------------- | ------ | -------- |
| 生产者 | 1. 使用同步发送。<br />2. 使用分布式事务消息 |        |          |
| 服务端 | 1. 使用同步刷盘策略<br />2. 使用同步复制机制 |        |          |
| 消费者 | 1. ACK确认机制                               |        |          |

## 消息重复: 幂等处理

[保证消息没有重复消费](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-that-messages-are-not-repeatedly-consumed.md)？

由消费者去保障消息重复问题，加操作表(数据库唯一索引或redis等)手段保障

## 消息积压

<aside> 💡 本着解决线上异常为**最高优先级**，然后通过监控和日志进行排查并优化业务逻辑，最后是扩容消费端和分片的数量。

</aside>

1. 保证消费者速度没问题，即先排查bug。
2. 扩容增加消费者
	1. 如rocketmq,默认是4个MessageQueue。如果当前queue<消费节点数量，则新增消费节点梳理
	2. 因queue不能动态新增，所以新增一个比较大queue的topic，将旧topic消息迁移到新的上来。
3. 提升消费进度
	1. 增加并发，如多线程等

[消息积压](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-that-messages-are-not-repeatedly-consumed.md)

### 提高消费并行度

绝大部分消息消费行为都属于 IO 密集型，即可能是操作数据库，或者调用 RPC，这类消费行为的消费速度在于后端数据库或者外系统的吞吐量，通过增加消费并行度，可以提高总的消费吞吐量，但是并行度增加到一定程度，反而会下降。所以，应用必须要设置合理的并行度。 如下有几种修改消费并行度的方法：

同一个 ConsumerGroup 下，通过增加 Consumer 实例数量来提高并行度（需要注意的是超过订阅队列数的 Consumer 实例无效）。可以通过加机器，或者在已有机器启动多个进程的方式。 提高单个 Consumer 的消费并行线程，通过修改参数 consumeThreadMin、consumeThreadMax 实现。

### 批量方式消费

某些业务流程如果支持批量方式消费，则可以很大程度上提高消费吞吐量，例如订单扣款类应用，一次处理一个订单耗时 1 s，一次处理 10 个订单可能也只耗时 2 s，这样即可大幅度提高消费的吞吐量，通过设置 consumer 的 consumeMessageBatchMaxSize 返个参数，默认是 1，即一次只消费一条消息，例如设置为 N，那么每次消费的消息数小于等于 N。

### 跳过非重要消息

发生消息堆积时，如果消费速度一直追不上发送速度，如果业务对数据要求不高的话，可以选择丢弃不重要的消息。例如，当某个队列的消息数堆积到 100000 条以上，则尝试丢弃部分或全部消息，这样就可以快速追上发送消息的速度。示例代码如下：

`public ConsumeConcurrentlyStatus consumeMessage( List<MessageExt> msgs, ConsumeConcurrentlyContext context) { long offset = msgs.get(0).getQueueOffset(); String maxOffset = msgs.get(0).getProperty(Message.PROPERTY_MAX_OFFSET); long diff = Long.parseLong(maxOffset) - offset; if (diff > 100000) { // TODO 消息堆积情况的特殊处理 return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; } // TODO 正常消费过程 return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; }`

### 优化每条消息消费过程

## 顺序性

[advanced-java/how-to-ensure-the-order-of-messages.md at main · doocs/advanced-java · GitHub](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-the-order-of-messages.md)

rocketMq如何保证顺序性

> [!TIP] 整体思路💡
>  1. 单一生产者和单一消费者，单一queue
