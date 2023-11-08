---
title: §redis
date created: 2023-11-01
date modified: 2023-11-08
---

+ 基础使用
	+ [[为什么用redis]]
+ 数据结构
	+ [[redis-数据结构]]
		+ 基础类型
			+ [[redis-string |String]]
			+ Hash
			+ [[redis-list |List]]
			+ Set
			+ Zset
		+ 特殊类型
			+ [[Hyperloglog]]
			+ [[redis-bitmap |bitmap]]
			+ [[redis-geo |geo]]
		+ [[redis-stream |stream]]
		+ 对象机制
			+ [[redisObject]]
			+ [[redis-对象共享]]
			+ [[reids-内存淘汰策略与删除 |对象淘汰]]
	+ [[redis-底层数据结构]]
+ 核心知识
	+ [[redis-内存管理]]
		+ [[reids-内存淘汰策略与删除]]
			+ [[redis-lru]] 最久未使用
			+ [[redis-lfu]] 最少使用
	+ [[reids-事务]]
	+ [[redis-订阅发布]]
+ 分布式  
	+ 高性能
		+ [[redis高性能]]
	+ 高可用
		+ [[ redis-持久化]] RDB/AOF/混合
		+ [[redis-数据复制]]
	+ 高扩展
		+ [[redis-集群]]
	+ 分布式常见问题
		+ [[redis-脑裂]]
+ 使用场景  
	+ [[缓存]]
	+ [[分布式锁]]  
	+ [[redis-消息队列]]  
	+ [[redis-search]]  
+ 优化案例
	+ [[redis-内存优化]]
	+ [[reids-选择合适数据结构，减少内存使用]]
