---
aliases:
  - AbstractQueuedSynchronizer
date created: 2022-09-26
date modified: 2022-09-26
title: AQS
---

> [!TIP] 技巧💡
> 1. AQS定义了我们实现锁的模板，具体实现由各个子类完成。
> 2. 内部实现了FIFO的队列(存储载体是Node)以及state状态变量。
> 3. 会把需要等待的线程以Node的形式放到这个先进先出的队列上，state变量则表示为当前锁的状态。
> 4. 先进先出队列存储的载体叫做Node节点，该节点标识着当前的状态值、是独占还是共享模式以及它的前驱和后继节点等等信息。

![](http://image.clickear.top/20220926171929.png)



1. 








## 资料

[AQS和ReentrantLock | 对线面试官](http://javainterview.gitee.io/luffy/2021/08/19/02-Java%E5%B9%B6%E5%8F%91/04.%20AQS%E5%92%8CReentrantLock/)
