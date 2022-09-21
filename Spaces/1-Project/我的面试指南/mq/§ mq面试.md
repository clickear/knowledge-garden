---
date created: 2022-09-19
date modified: 2022-09-19
title: § mq面试
---

+ 什么是消息队列？（作用）
 + 优点:
	 + [[异步]]
	 + [[削峰]]
	 + [[解耦]]
 + 缺点:
	 + 可用性低，要考虑丢失、重复消费等
	 + 复杂度高
	 + 上下游数据一致性问题
+ 消息队列技术选型？
	+ rabbitmq,社区不活跃
	+ activemq, 社区丰富，吞吐量低。elang语言
	+ rocketmq，阿里捐赠给apache，吞吐量高。topic时，性能不会大幅度下降？
	+ kafka， 基于大数据领域。日志和实时计算用的多。 topic多时，性能差
+ 高可用
	+ 存储结构
		+ [[kafka]], 每个partition是一个文件。会造成磁盘竞争IO
		+ rocketMq,采用commitlog + comsume queue的方式。相当于commitlog存储全部数据，queue存储偏移量。
	+ 数据分片存储
		+ kafka一个topic下面的所有消息都是以partition的方式分布式的存储在多个节点上。即每一个机器，都没有完整的topic数据
		+ 一个 Topic 分布在多个Broker 上，一个 Broker 可以配置多个 Topic，它们是多对多的关系。如果某个 Topic 消息量很大，应该给它多配置几个队列，并且尽量多分布在不同 Broker 上，以减轻某个 Broker 的压力
	+ 主节点选举
		+ rocketmq，基于raft进行选举，超过一半成功即可。在写数据时，只需一半成功即可。
		+ kafka，是基于zk进行选择的。在写数据时，可配置多少个副本写入成功即算成功
+ [[消息丢失]](以rocketmq为例)
	+ 生产者: 同步发送，或者分布式事务消息
	+ 服务端:
		+ 持久化: 同步刷盘策略
		+ 主从同步: 同步复制
	+ 消费者:
		+ ACK确认机制
+ [[重复消费]](幂等处理)
	+ 分布式锁
	+ 数据库唯一索引
+ [[消息堆积]]
	+ 1. 优先保证线上，先处理bug，保证消费者没问题
	+ 2. 提升消费者能力
		+ 多线程并发处理
		+ 新增消费节点
			+ 如果queue > 消费节点数，增加消费节点
			+ 如果queue < 消费节点数，
				+ 新增topic，并设置queue大小为消费节点数量
				+ 将旧的toppic数据，消费重新发送到新的topic进行消费
		+ 批量消费
		+ 跳过不重要消息，记录后续处理
		+ 优化代码
+ 使用case:
	+ 大事务拆成 小事务 + 异步化处理。 omq备货流程优化
	+ 异步处理, 工作流发送消息
	+ 延迟消息，超时未支付
+ 优化case:
	+ 抖音接口限流，导致重试调用失败
		+ 多线程调用
		+ 增加消费者
+ [如何设计mq](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/mq-design.md)
	+ 可伸缩性，数据分片存储。broker -> topic -> partition，每个 partition 放一个机器，就存一部分数据
	+ 持久化，顺序写。
		+ kafaka, partition写在一个文件中
		+ rocketmq， 元数据都写在commitlog中，queue写偏移量
	+ 可用性
		+ leader选举
		+ 多副本，主从

#todo/continue  
[29 从 RocketMQ 学 Netty 网络编程技巧.md](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/RocketMQ%20%E5%AE%9E%E6%88%98%E4%B8%8E%E8%BF%9B%E9%98%B6%EF%BC%88%E5%AE%8C%EF%BC%89/29%20%E4%BB%8E%20RocketMQ%20%E5%AD%A6%20Netty%20%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B%E6%8A%80%E5%B7%A7.md)

[可靠分布式系统-paxos的直观解释 - OpenACID Blog](https://blog.openacid.com/algo/paxos/)

[源码分析RocketMQ系列索引_中间件兴趣圈的博客-CSDN博客_rocketmq 索引](https://blog.csdn.net/prestigeding/article/details/78888290)