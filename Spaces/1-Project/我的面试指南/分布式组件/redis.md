---
mindmap-plugin: basic
date created: 2022-09-15
date modified: 2023-09-08
title: redis
---

# redis

## 数据类型

命名前缀:

> [!TIP] 技巧💡
>  + Set commands start with `s`
>  + Hash commands start with `h`
>  + List commands start with `l`
>  + Sorted set commands start with `z`.[^1](为什么是z前缀
>  + Stream commands start with `x`
>  + Hyperloglog commands start with `pf` [^2]

| 类型   | 作用 | 命名 |     |
|:------ |:---- |:---- |:--- |
| String |  value: 可以是字符串、数值、浮点数。支持自增、自减操作   | get: 获取<br> set: 设置 <br> del: 删除，所有类型都不支持  |     |
| List   |  数组，类比ArrayList    | lpush lpop lrange lindex rpush rpop  |     |
| Set    |  set,类比HashSet    |   sadd,SMEMBERS, SISMEMBER, SREM   |     |
| Hash   |  map, 类比hashmap    |   HSET，HGET, HGETALL, HDEL   |     |
| ZSET   |  有序的set,类比 TreeSet  |   ZADD, ZRANGE, ZRAGEBYSCORE, ZREM   |     |
|        |      |      |     |

## 高性能

### 线程模型（核心读写操作 | 单线程 持久化、集群同步、处理网络请求 | 多线程）

Redis 的网络 I/O 线程，以及键值的 SET 和 GET 等读写操作都是由一个线程来完成的。但 **Redis 的持久化、集群同步**等操作，则是由另外的线程来执行的。

虽然 Redis 一直是单线程模型，但是在 Redis 6.0 版本之后，也采用了多个 I/O 线程来处理网络请求，这是因为随着网络硬件的性能提升，Redis 的性能瓶颈有时会出现在网络 I/O 的处理上，所以为了提高网络请求处理的并行度，Redis 6.0 对于网络请求采用多线程来处理。但是对于读写命令，Redis 仍然使用单线程来处理

#### 快的原因

1. 纯内存操作、高效的数据结构
2. 单线程，节省上下文切换和锁
3. Redis 采用了 I/O 多路复用机制处理大量的客户端 Socket 请求，这让 Redis 可以高效地进行网络通信. reactor

## 高可用

### 持久化(RDB + AOF)

缓存数据库的读写都是在内存中，所以它的性能才会高，但在内存中的数据会随着服务器的重启而丢失，为了保证数据不丢失，要把内存中的数据存储到磁盘，以便缓存服务器重启之后，还能够从磁盘中恢复原有的数据，这个过程就是 Redis 的数据持久化

Redis 的数据持久化有三种方式

#### AOF 日志（Append Only File，文件追加方式）

<aside> 💡 记录所有的操作命令，并以文本的形式追加到文件中。

</aside>

通常情况下，关系型数据库（如 MySQL）的日志都是“写前日志”（Write Ahead Log, WAL），也就是说，在实际写数据之前，先把修改的数据记到日志文件中，以便当出现故障时进行恢复，比如 MySQL 的 redo log（重做日志），记录的就是修改后的数据。

而 AOF 里记录的是 Redis 收到的每一条命令，这些命令是以文本形式保存的，不同的是，Redis 的 AOF 日志的记录顺序与传统关系型数据库正好相反，它是写后日志，“写后”是指 Redis 要先执行命令，把数据写入内存，然后再记录日志到文件。

![AOF 执行过程](http://image.clickear.top/20211222184311.png)

AOF 执行过程

那么面试的考察点来了：**Reids 为什么先执行命令，在把数据写入日志呢**？为了方便你理解，我整理了关键的记忆点：

- 因为 ，Redis 在写入日志之前，不对命令进行语法检查；
- 所以，只记录执行成功的命令，避免了出现记录错误命令的情况；
- 并且，在命令执行完之后再记录，不会阻塞当前的写操作。

当然，这样做也会带来风险（这一点你也要在面试中给出解释）。

- **数据可能会丢失：** 如果 Redis 刚执行完命令，此时发生故障宕机，会导致这条命令存在丢失的风险。
- **可能阻塞其他操作：** 虽然 AOF 是写后日志，避免阻塞当前命令的执行，但因为 AOF 日志也是在主线程中执行，所以当 Redis 把日志文件写入磁盘的时候，还是会阻塞后续的操作无法执行

#### **RDB 快照（Redis DataBase）**：

<aside> 💡 将某一个时刻的内存数据，以二进制的方式写入磁盘。

</aside>

redis 增加了 RDB 内存快照（所谓内存快照，就是将内存中的某一时刻状态以数据的形式记录在磁盘中）的操作，它即可以保证可靠性，又能在宕机时实现快速恢复

##### **RDB 做快照时会阻塞线程吗?会，类似jvm stw**

因为 Redis 的单线程模型决定了它所有操作都要尽量避免阻塞主线程，所以对于 RDB 快照也不例外，这关系到是否会降低 Redis 的性能。

- save 命令在主线程中执行，会导致阻塞
- bgsave 命令则会创建一个子进程，用于写入 RDB 文件的操作，避免了对主线程的阻塞，这也是 Redis RDB 的默认配置

##### **RDB 做快照的时候数据能修改吗？**

很明显，在做快照的时候，做修改会导致数据不一致。处理方案也很简单，就是把修改的数据，同步一份给bgsave进程处理就好。

![http://image.clickear.top/20211222184717.png](http://image.clickear.top/20211222184717.png)

#### **混合持久化方式(游戏中的暂存点功能)**

<aside> 💡 Redis 4.0 新增了混合持久化的方式，集成了 RDB 和 AOF 的优点

</aside>

Redis 对 RDB 的执行频率非常重要，因为这会影响快照数据的完整性以及 Redis 的稳定性。Redis 4.0 后，**增加了 AOF 和 RDB 混合的数据持久化机制：** 把数据以 RDB 的方式写入文件，再将后续的操作命令以 AOF 的格式存入文件，既保证了 Redis 重启速度，又降低数据丢失风险。

### 数据复制（副本或者分片）

从 Redis 的多服务节点来考虑，比如 Redis 的主从复制、哨兵模式，以及 Redis 集群。

#### 主从同步 (主从复制)，无法自动恢复，难在线扩容

这是 Redis 高可用服务的最基础的保证，实现方案就是将从前的一台 Redis 服务器，同步数据到多台从 Redis 服务器上，即一主多从的模式，这样我们就可以对 Redis 做读写分离了，来承载更多的并发操作，这里和 MySQL 的主从复制原理上是一样的。  
缺点:

1. 自动容错和恢复功能.需要人工介入
2. 难维护，无法在线扩容

#### Sentinel（哨兵模式）自动版主从复制。副本存储

在使用 Redis 主从服务的时候，会有一个问题，就是当 Redis 的主从服务器出现故障宕机时，需要手动进行恢复，为了解决这个问题，Redis 增加了哨兵模式（因为哨兵模式做到了可以监控主从服务器，并且提供自动容灾恢复的功能）

master选举过程:  
哨兵watch监控，标记 主观下线，然后进行投票。投票超过半数才算通过客观下线。

![http://image.clickear.top/20211222185049.png](http://image.clickear.top/20211222185049.png)

#### **Redis Cluster（集群）**

Redis Cluster 是一种分布式去中心化的运行模式，是在 Redis 3.0 版本中推出的 Redis 集群方案，它将数据分布在不同的服务器上，以此来降低系统对单主节点的依赖，从而提高 Redis 服务的读写性能。

Redis Cluster 方案采用哈希槽（Hash Slot），来处理数据和实例之间的映射关系。在 Redis Cluster 方案中，一个切片集群共有 16384 个哈希槽，这些哈希槽类似于数据分区，每个键值对都会根据它的 key，被映射到一个哈希槽中.

![http://image.clickear.top/20211222185255.png](http://image.clickear.top/20211222185255.png)

## 使用场景

### 缓存

<aside> 💡 很多性能问题，可以增加缓存来处理。比如处理器，增加一级、二级缓存加快访问速度。

</aside>
[[缓存]]

### 分布式锁CP

[[分布式锁]]

## 队列

[[MQ]]

## 资料

- [](https://book.clickear.top/%E6%8B%89%E5%8B%BE_%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1%E9%9D%A2%E8%AF%95%E7%B2%BE%E8%AE%B2/13%20%7C%20%E7%BC%93%E5%AD%98%E5%8E%9F%E7%90%86%EF%BC%9A%E5%BA%94%E5%AF%B9%E9%9D%A2%E8%AF%95%E4%BD%A0%E8%A6%81%E6%8E%8C%E6%8F%A1%20Redis%20%E5%93%AA%E4%BA%9B%E5%8E%9F%E7%90%86%EF%BC%9F.html)[https://book.clickear.top/拉勾_架构设计面试精讲/13](https://book.clickear.top/%E6%8B%89%E5%8B%BE_%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1%E9%9D%A2%E8%AF%95%E7%B2%BE%E8%AE%B2/13) | 缓存原理：应对面试你要掌握 Redis 哪些原理？.html
- [](https://book.clickear.top/%E6%8B%89%E5%8B%BE_%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1%E9%9D%A2%E8%AF%95%E7%B2%BE%E8%AE%B2/14%20%7C%20%E7%BC%93%E5%AD%98%E7%AD%96%E7%95%A5%EF%BC%9A%E9%9D%A2%E8%AF%95%E4%B8%AD%E5%A6%82%E4%BD%95%E5%9B%9E%E7%AD%94%E7%BC%93%E5%AD%98%E7%A9%BF%E9%80%8F%E3%80%81%E9%9B%AA%E5%B4%A9%E7%AD%89%E9%97%AE%E9%A2%98%EF%BC%9F.html)[https://book.clickear.top/拉勾_架构设计面试精讲/14](https://book.clickear.top/%E6%8B%89%E5%8B%BE_%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1%E9%9D%A2%E8%AF%95%E7%B2%BE%E8%AE%B2/14) | 缓存策略：面试中如何回答缓存穿透、雪崩等问题？.html

[^1]: [python - Why Redis 'Zset' means 'Sorted Set'? - Stack Overflow](https://stackoverflow.com/questions/64020570/why-redis-zset-means-sorted-set)

[^2]: [初识Redis的数据类型HyperLogLog-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1650031)
