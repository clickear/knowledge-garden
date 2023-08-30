---
title: access method
date created: 2023-06-19
date modified: 2023-08-30
---

> [!TIP] 技巧💡  
> access method ,是我们对数据库数据进行读或者写的方式。 数据结构无处不在。
>

+ 数据组织：如何将这些数据结构合理地放入 memory/pages 中，以及为了支持更高效的访问，应当存储哪些信息。

- 并发性：如何支持数据的并发访问。  
在DBMS中，最重要的就是 [[哈希表]] 和[[trees]]

常用的数据存储，我们一般分为2类。

+ [[Tuple-oriented]]存数据，使用[[In-place update structure]]就地更新结构，如[[B+树]]。是直接覆盖旧记录来存储更新内容。
+ [[Log-structured]]存日志，使用[[out-place-update-structure]]异地更新结构，会将更新的内容存到到新的位置，而不是覆盖。

|流派|主要特点|基本思想|代表|
|---|---|---|---|
|log-structured 流|只允许追加，所有修改都表现为文件的追加和文件整体增删|变随机写为顺序写|Bitcask、LevelDB、RocksDB、Cassandra、Lucene|
|update-in-place 流|以页（page）为粒度对磁盘数据进行修改|面向页、查找树|B 族树，所有主流关系型数据库和一些非关系型数据库|

## 结构

| 结构          | 主义特点                                         |                基本思想                |  主要问题   | 优化思路 |
|:------------- |:------------------------------------------------ |:-------------------------------------- | --- | --- |
| [[哈希表]]    | 主要适用于pagetable,自适应hash等                 | 通过hash函数，查找很快 |   不支持范围查询。如[[mysiam]]存储引擎，hash冲突  | 解决hash冲突，如拉链法等|
| [[Trees]] B树 | 就地更新数据，直接覆盖。以页的力度对磁盘进行修改.主流关系型数据库的选择 |         增加读性能，已经是排序好的最好数据。                   |   写的时候可能会存在页合并，页分裂等情况。并发控制不好  |B link树，解决[[B+树]]并发控制自下而上的问题|
| [[LSM树]]              |   [[Log-structured]]的代表，数据通过日志的方式进行存储。KV数据库、大数据相关的数据库的选择.LSM-Tree 并不是一种严格的树结构，而是一种**内存+磁盘的多层存储结构**。HBase、LevelDB、RocksDB这些 NoSQL 存储都使用了 LSM-Tree    | 增加写性能，通过随机写变顺序写(写日志很快) | 读的时候，获取最新数据需要根据日志进行推演                                          | 1. 优化SSTABLE查找 [[Bloom过滤器]]2. 层级SSTABLE，进行[[compaction]]合并 |                                       |     |

## 资料

[DDIA 读书笔记（三）：B-Tree 和 LSM-Tree | 木鸟杂记](https://www.qtmuniao.com/2022/04/16/ddia-reading-chapter3-part1/)

[B+树,B-link树,LSM树...一个视频带你了解常用存储引擎数据结构（合集）\_哔哩哔哩\_bilibili](https://www.bilibili.com/video/BV1se4y1U7Dn/?buvid=XU395E039B5DE558C9710D2DB3A55AC3F6F33&is_story_h5=false&mid=H0ejNYFOat6NVNeix6q0Vw%3D%3D&p=1&plat_id=114&share_from=ugc&share_medium=android&share_plat=android&share_session_id=73a47f18-5673-4ea9-8970-0fbbd9b6b0de&share_source=WEIXIN&share_tag=s_i&timestamp=1686585416&unique_k=qp21WwT&up_id=61981458)

[简述LSM-Tree - pedro7 - 博客园](https://www.cnblogs.com/WangXianSCU/p/15939129.html)
