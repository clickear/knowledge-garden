---
title: §redis
date created: 2023-11-01
date modified: 2023-11-23
---

+ 基础使用
	+ [[为什么用redis]]
+ 数据结构
	+ [[redis-数据结构]]
		+ 基础类型
			+ [[redis-string |String]]
			+ Hash [[渐进式rehash]]
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
	+ [[redis-阻塞点]]
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
		+ [[序列化器]] + [[压缩算法]]
	+ [[reids-选择合适数据结构，减少内存使用]]
	+ [[redis-开发规范]]
+ 解决方案
	+ [[✨分布式多级缓存]]
	+ [[✨ redis-集群方案]]
	+ [[✨ redis-延迟队列]]
+ 学习资料
	+ [Redis 核心技术与实战](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Redis%20%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF%E4%B8%8E%E5%AE%9E%E6%88%98) ⭐
	+ [Redis 源码剖析与实战](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Redis%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98)
	+ [Redis 核心原理与实战](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/Redis%20%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%98)
+ 运维
	+ [[redis-运维工具]]
	+ [[redis-如何排查bigkey]]
	+ [[redis-如何排查慢查询]]
