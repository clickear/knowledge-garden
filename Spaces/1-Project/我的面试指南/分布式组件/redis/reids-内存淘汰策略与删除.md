---
title: reids-内存淘汰策略与删除
date created: 2023-11-08
date modified: 2023-11-28
---

> [!TIP] 删除策略，是指到了过期时长未删除的💡  
> 惰性删除: 是指不主动删除，只有在访问key时，都会判断key是否过期。如果过期则删除  
> 定期删除: 是指每隔一段时间，随机获取一些key，判断是否过期，过期则删除。  
> -- 为避免随机获取时，执行太长，或者执行循环检查。会限定执行时长，判断删除key占比>25%

> [!TIP] 内存淘汰(Redis 的运行内存达到了某个阀值，就会触发**内存淘汰机制**)💡  
> redis，需要配置了maxmemory 最大使用内存时。并且超过了这个值，才会进行内存淘汰。

## 删除了数据是立即释放吗？

删除了数据，并不是立即释放的。  
[[redis-内存管理]]

## 内存淘汰-maxmemory

在 64bit 系统下，`maxmemory` 设置为 0 表示不限制 Redis 内存使用，在 32bit 系统下，`maxmemory` 隐式不能超过 3GB。 maxmemory_policy,淘汰策略。

``` c
// server.c
if (server.arch_bits == 32 && server.maxmemory == 0) {
	serverLog(LL_WARNING,"Warning: 32 bit instance detected but no memory limit set. Setting 3 GB maxmemory limit with 'noeviction' policy now.");
	server.maxmemory = 3072LL*(1024*1024); /* 3 GB */
	server.maxmemory_policy = MAXMEMORY_NO_EVICTION;
}
```

## 淘汰策略

+ _**不进行数据淘汰的策略**_
	+ **noeviction**（Redis3.0之后，默认的内存淘汰策略） ：它表示当运行内存超过最大设置内存时，不淘汰任何数据，而是不再提供服务，直接返回错误。
+ _**进行数据淘汰的策略**_
	+ 在设置了过期时间的数据中淘汰
		+ **volatile-random**：随机淘汰设置了过期时间的任意键值；
		+ **volatile-ttl**：优先淘汰更早过期的键值。
		+ **volatile-lru**（Redis3.0 之前，默认的内存淘汰策略）：淘汰所有设置了过期时间的键值中，最久未使用的键值；
		+ **volatile-lfu**（Redis 4.0 后新增的内存淘汰策略）：淘汰所有设置了过期时间的键值中，最少使用的键值；
	+ 所有数据范围
		+ **allkeys-random**：随机淘汰任意键值;
		+ **allkeys-lru**：淘汰整个键值中最久未使用的键值
		+ **allkeys-lfu**（Redis 4.0 后新增的内存淘汰策略）：淘汰整个键值中最少使用的键值。

### LRU和[[LFU]]

**[[LRU]]** 全称是 Least Recently Used 翻译为**最近最少使用**，会选择淘汰最近最少使用的数据  
[[redis-lru]]，每个对象记录最后访问时间，淘汰时，随机获取数据，淘汰最久没有使用的数据。  
[[redis-lfu]], 每个对象记录最后访问时间戳和访问频次。
