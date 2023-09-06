---
title: DoubleWrite
date created: 2023-09-06
date modified: 2023-09-06
---

> [!TIP] 技巧💡  
>  [[DoubleWrite]],即双写。先写副本(DoubleWrite buffer)，在写到磁盘中的数据页
> 1. mysql将脏页flush到磁盘，基本单位是页，每页的大小是16k，而linux操作系统的page是4k。就存在一种部分写入成功的时候，crash了，导致磁盘的数据不完整。
> 2. redolog的重放功能，是基于完整的数据页，由于磁盘页中的数据已经不完整，没有办法进行重放了。  
> 解决思路:
> 1. 新增一个"中间存储媒介"-副本，先写入副本成功之后，才写入到磁盘中。
>

## 为什么需要这个结构？

每次操作的单位不一样，mysql页单位是16k，而操作系统每次操作是4k，导致每次write页，不是原子性。即存在部分写入成功的情况。这样会导致，磁盘中的数据页不是完整的。那么肯定会想到，重启后，通过redolog来恢复内存数据页的数据即可？但是这个是不行的，因为redolog是要根据完整的数据页才可以。此时磁盘中的页已经不是完整的了。

> [!ABSTRACT] 摘要  
>  主要思想，就是 保证无论什么时候，都有一个地方的数据是完整的。即新增一个副本即可。

![image.png](http://image.clickear.top/20230906105954.png)

## [[DoubleWrite]] 流程

**Doublewrite Buffer，它与传统的“Buffer”又不同，它分为内存和磁盘的两层架构。（传统的“Buffer”，大部分是内存存储；而DWB里的数据，是需要落地的）**

1. 为什么需要落地？因为需要根据副本进行中间存储。保障任何时刻，都有一个地方，数据是完整的。  
![image.png](http://image.clickear.top/20230906110433.png)  
注意，mysql8.0的时候，doublewrite buffer，不放在系统表空间。而是一个单独的文件。

**第1步：页数据先memcopy到DWB的内存里；**  
**第2步：DWB的内存里的数据页，会先刷到DWB的磁盘上；**  
**第3步：DWB的内存里的数据页，再刷到数据磁盘存储.ibd文件上；**  
**备注：DWB内存结构由128个页（Page）构成，所以容量只有：16KB × 128 = 2MB。**

双写，也就是 第2/3步。

### 如何保障无论何时，都有一个地方存在完整的数据？

如果在第1、2步，出现宕机，此时磁盘还是完整的数据，根据redolog就可以重放到数据了。  
如果在第3步，出现宕机，此次根虽然磁盘的数据不是完整的，但是DWB的数据是完整的。直接根据DBW来恢复即可。

### 双写，是否会有性能问题

1. 第一步，使用memcopy，性能很快。
2. 第二步，**DWB的内存fsync刷到DWB的磁盘，属于顺序追加写，速度也很快**。
3. 第三部，**刷磁盘，随机写，本来就需要进行，不属于额外操作；**

## 扩展

### redolog，需要[[DoubleWrite]]吗？

redolog不需要，因为它的单位是512b。不存在部分写入的情况。

### 其它数据库，有dw吗？

4kB: sqllite, oracle 8KB: sqlserver/postgresql 16kB。  
如果mysql的每个数据页大小设置成4k的话，也可以不用dw

## 存在疑惑

1. [[DoubleWrite]]的工作流程中，为什么需要第一步？即为什么需要内存中的DWB

## 参考文章

[MySQL各种“Buffer”之Doublewrite Buffer - 墨天轮](https://www.modb.pro/db/114783)  
[InnoDB关键特性之double write - GeaoZhang - 博客园](https://www.cnblogs.com/geaozhang/p/7241744.html)
