---
title: §redis
date created: 2023-11-01
date modified: 2023-11-01
---

+ 基础使用
	+ [[为什么用redis]]
	+ [[redis为什么快]]
+ 数据结构
	+ 数据类型
		+ 基础类型
			+ String
			+ Hash
			+ List
			+ Set
			+ Zset
		+ 特殊类型
			+ [[Hyperloglog]]
			+ [[redis-bitmap]]
			+ [[redis-geo]]
		+ [[redis-stream]]
		+ 对象机制
			+ [[redisObject]]
			+ 对象共享
			+ 对象淘汰
	+ [[redis-底层数据结构]]
+ 核心知识
	+ [[ redis-持久化]] RDB/AOF/混合
	+ 内存淘汰策略
	+ [[reids-事务]]
	+ [[redis-订阅发布]]
+ 分布式  
	+ 高可用
		+ [[redis-主从复制]]
	+ 高扩展
		+ [[redis-集群]]
+ 使用场景  
	+ [[redis-分布式锁]]  
	 [[reids-消息队列]]  
	 [[redis-search]]

+ 优化案例
	+ [[reids-选择合适数据结构，减少内存使用]]
