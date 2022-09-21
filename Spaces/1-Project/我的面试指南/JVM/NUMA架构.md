---
date created: 2022-09-05
date modified: 2022-09-05
title: NUMA架构
---

![](http://image.clickear.top/20210913171600.png)

NUMA对应的有UMA，UMA即Uniform Memory Access Architecture，NUMA就是Non Uniform Memory Access Architecture。UMA表示内存只有一块，所有CPU都去访问这一块内存，那么就会存在竞争问题（争夺内存总线访问权），有竞争就会有锁，有锁效率就会受到影响，而且CPU核心数越多，竞争就越激烈。NUMA的话**每个CPU对应有一块内存**，且这块内存在主板上离这个CPU是*最近的*，每个CPU优先访问这块内存，那效率自然就提高了
