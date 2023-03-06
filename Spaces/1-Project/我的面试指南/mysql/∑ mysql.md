---
date created: 2022-09-07
date modified: 2023-03-06
title: ∑ mysql
---

## ::一览mysql

[mysql架构图](https://www.processon.com/diagraming/6098b801637689782cc1f2d4)：  

理解change buffer还得先理解buffer pool是啥，顾名思义，硬盘在读写速度上相比内存有着数量级差距，如果每次读写都要从磁盘加载相应数据页，DB的效率就上不来，因而为了化解这个困局，几乎所有的DB都会把缓存池当做标配（在内存中开辟的一整块空间，由引擎利用一些命中算法和淘汰算法负责维护和管理），change buffer则更进一步，把在内存中更新就能可以立即返回执行结果并且满足一致性约束（显式或隐式定义的约束条件）的记录也暂时放在缓存池中，这样大大减少了磁盘IO操作的几率。  
因为InnoDB按数据页为单元进行读写。对于更新语句来说，对来内存中的数据直接在内存更新，对于不在内存中的数据，对于唯一索引，还是需要进行读内存，但对于普通索引就可以通过使用change buffer来减少对磁盘的访问

![](http://image.clickear.top/mysql%E6%9E%B6%E6%9E%84%E5%9B%BE_%E6%95%B4%E7%90%86%E7%89%88.png)

[mysql思维导图](https://www.processon.com/mindmap/6096150f637689782cbc516d)：

![](http://image.clickear.top/mysql.png)

## :: 原资料地址

[mysql思维导图](https://www.processon.com/mindmap/6096150f637689782cbc516d)：

![](http://image.clickear.top/mysql.png)

## :: 原资料地址
