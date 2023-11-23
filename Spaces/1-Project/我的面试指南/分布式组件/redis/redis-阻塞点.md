---
title: redis-阻塞点
date created: 2023-11-23
date modified: 2023-11-23
---

## checklist

1. 使用复杂度过高的命令或一次查询全量数据；
2. 操作 bigkey；
3. 大量 key 集中过期；
4. 内存达到 maxmemory；
5. 客户端使用短连接和 Redis 相连；
6. 当 Redis 实例的数据量大时，无论是生成 RDB，还是 AOF 重写，都会导致 fork 耗时严重；
7. AOF 的写回策略为 always，导致每个操作都要同步刷回磁盘；
8. Redis 实例运行机器的内存不足，导致 swap 发生，Redis 需要到 swap 分区读取数据；
9. 进程绑定 CPU 不合理；
10. Redis 实例运行机器上开启了透明内存大页机制；
11. 网卡压力过大。

## 阻塞点

![redis阻塞点.jpg](http://image.clickear.top/redis%E9%98%BB%E5%A1%9E%E7%82%B9.jpg)

## 资料

+ [16 异步机制：如何避免单线程模型的阻塞？](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Redis%20%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/16%20%20%e5%bc%82%e6%ad%a5%e6%9c%ba%e5%88%b6%ef%bc%9a%e5%a6%82%e4%bd%95%e9%81%bf%e5%85%8d%e5%8d%95%e7%ba%bf%e7%a8%8b%e6%a8%a1%e5%9e%8b%e7%9a%84%e9%98%bb%e5%a1%9e%ef%bc%9f.md)
+ [19 波动的响应延迟：如何应对变慢的Redis？（下）](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Redis%20%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/19%20%20%e6%b3%a2%e5%8a%a8%e7%9a%84%e5%93%8d%e5%ba%94%e5%bb%b6%e8%bf%9f%ef%bc%9a%e5%a6%82%e4%bd%95%e5%ba%94%e5%af%b9%e5%8f%98%e6%85%a2%e7%9a%84Redis%ef%bc%9f%ef%bc%88%e4%b8%8b%ef%bc%89.md)
