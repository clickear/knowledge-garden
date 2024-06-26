---
title: redis-脑裂
date created: 2023-11-05
date modified: 2023-11-05
---

[[脑裂]]，是一个分布式中常见的问题。

## redis如何发送脑裂

> [!TIP] 技巧💡  
> 简单来说，就是redis主库的假故障(如cpu飙高，没想要从库的心跳请求，被哨兵判定为了故障。但是实际上是能处理客户端写请求)。从库升级了主库，此时就发送了脑裂。

![image.png](http://image.clickear.top/20231105013055.png)

### 脑裂带来的影响（数据丢失）

> [!TIP] 技巧💡  
> 在主从切换的过程中，如果原主库只是“假故障”，它会触发哨兵启动主从切换，一旦等它从假故障中恢复后，又开始处理请求，这样一来，就会和新主库同时存在，形成脑裂。等到哨兵让原主库和新主库做**全量同步**后，原主库在切换期间保存的数据就丢失了。

![image.png](http://image.clickear.top/20231105013028.png)

## 资料

[33 脑裂：一次奇怪的数据丢失](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Redis%20%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/33%20%20%e8%84%91%e8%a3%82%ef%bc%9a%e4%b8%80%e6%ac%a1%e5%a5%87%e6%80%aa%e7%9a%84%e6%95%b0%e6%8d%ae%e4%b8%a2%e5%a4%b1.md)
