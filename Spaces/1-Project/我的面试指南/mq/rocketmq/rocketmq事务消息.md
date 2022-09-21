---
date created: 2022-09-19
date modified: 2022-09-19
title: rocketmq事务消息
---

> [!TIP] 技巧💡  
> 1. 解决的是**本地事务的执行和发消息这两个动作满足事务**的约束。
> 2. 通过2阶段提交来做。
> 	1. client端先发送一个half消息(这个半消息是指消费者消费不到，还未在broker提交)
> 	2. broker接收到半消息之后，将其放在队列为0的topic中。
> 	3. client端，提交本地事务。
> 	4. client端，通过oneway方式来发送事务结果
> 	5. broker根据结果，来处理半消息
> 	6. broker端，还有个反查机制。避免oneway方式因网络抖动等发送给broker失败

![](http://image.clickear.top/20220919093345.png)

## 资料

[Fetching Title#hp97](https://juejin.cn/post/6867040340797292558)
