---
date created: 2022-09-07
date modified: 2022-09-07
title: changebuffer
---

> [!INFO] 概念，channgebuffer,减少IO次数。  
>  它是一种应用在非唯一普通索引页(non-unique secondary index page)不在缓冲池中，对页进行了写操作，并不会立刻将磁盘页加载到缓冲池，而仅仅记录缓冲变更(buffer changes)，等未来数据被读取时，再将数据合并(merge)恢复到缓冲池中的技术  
适用于未加载到buffer pool 并且非唯一索引页

## ::我的理解

简单来说，就是不直接写到磁盘，先写到日志[[顺序读写]]很快。为保证加载到[[bufferpool]]内存数据的准确性，就需要在后续读取时，进行merge操作。

### merge操作(磁盘的数据不是最新的，需要与changebuffer进行合并恢复到缓冲池中)

1. 原始数据页加载到 Buffer Pool 时
2. 系统后台定时触发 merge 操作
3. MySQL 数据库正常关闭时。

### 持久化操作

触发持久化(Change Buffer 是 Buffer Pool 中的一部分,System Tablespace 中可以看到持久化 Change Buffer 的空间)

1. 有一个后台线程，会认为数据库空闲时；
2. 数据库缓冲池不够用时；
3. 数据库正常关闭时
4. redolog写满

## 突然宕机，changbuffer的数据怎么办？根据redolog来找回

[[redolog]]就是为了保证事务的持久性。因为change buffer是存在内存中的，万一机器重启，change buffer中的更改没有来得及更新到磁盘，就需要根据redo log来找回这些更新。 优点是减少磁盘I/O次数，即便发生故障也可以根据redo log来将数据恢复到最新状态。 缺点是会造成内存脏页，后台线程会自动对脏页刷盘，或者是淘汰数据页时刷盘，此时收到的查询请求需要等待，影响查询

## 适用场景(写多读少，这样少merge)

适合写多读少的场景，因为这样即便立即写了，也不太可能会被访问到，延迟更新可以减少磁盘I/O，只有普通索引会用到，因为唯一性索引，在更新时就需要判断唯一性，所以没有必要

## 参数

1. innodb_change_buffer_max_size(占changbuffer最大大小 默认25% 最大 50%)
2. innodb_change_buffering（哪些操作启用）

## ::QA

1. 为什么changebuffer可以减少IO次数？都是写磁盘有什么区别吗？
   > 诀窍在于，写日志记录，是[[顺序读写]]，速度很快。而磁盘操作，是[[随机读书]]
2. 为什么唯一索引用不了[[changebuffer]]?
   > 因为唯一索引，需要在更新时判断唯一性，就没必要多次一举。判断唯一性要先加载到内存中。那么就没办法直接把变更写到日志中
