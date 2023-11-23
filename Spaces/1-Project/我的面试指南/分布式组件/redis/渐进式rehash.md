---
title: 渐进式rehash
date created: 2023-11-23
date modified: 2023-11-23
---

![redisdb结构.jpg](http://image.clickear.top/redisdb%E7%BB%93%E6%9E%84.jpg)

redis 全局实例，定义了一个 dict实例。其中会有2个dicttable。[[redisObject]]

redis解决[[hash冲突]]，是通过数组+链表的方式。因为所有的数据，都是放在这个全局dict中。如果需要扩容，直接将原有数据全部复制到新的dicttable中，会导致性能很慢。  
如果需要扩容，设计到2部分数据。  
历史数据迁移和新插入数据。

1. 历史数据迁移，不是一次性迁移，而是每次迁移，这就是渐进式的由来。根据rehashidx,来获取当前迁移的进度。
2. 新数据，直接放到新的dicttable中即可。  
查询的时候，直接先从dicttable[0]查询，如果没空，在到新的dicttable[1]查询。

## 资料

[02 数据结构：快速的Redis有哪些慢操作？](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Redis%20%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/02%20%20%e6%95%b0%e6%8d%ae%e7%bb%93%e6%9e%84%ef%bc%9a%e5%bf%ab%e9%80%9f%e7%9a%84Redis%e6%9c%89%e5%93%aa%e4%ba%9b%e6%85%a2%e6%93%8d%e4%bd%9c%ef%bc%9f.md)
