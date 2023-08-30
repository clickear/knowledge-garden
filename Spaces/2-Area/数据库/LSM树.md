---
title: LSM树
date created: 2023-06-13
date modified: 2023-08-30
---
 > [!TIP] Log-structured Merge tree💡  
>  LSM-Tree 并不是一种严格的树结构，而是一种**内存+磁盘的多层存储结构**。HBase、LevelDB、RocksDB这些 NoSQL 存储都使用了 LSM-Tree.核心思想是利用**磁盘顺序写性能远高于随机写**

概念:

- **读放大。**读取数据时实际读取的数据量大于真正的数据量。例如在不同层次的 Table 中查找。
- **写放大。**写入数据时实际写入的数据量大于真正的数据量。例如写入时触发 Compact，导致写入大量数据。
- **空间放大。**数据占用的磁盘空间比真正的数据大小要大很多。即冗余存储。

LSM树，在 1（读写性能）的性能不是特别好。主要是通过compaction来操作整理LSM树的结构。

![image.png](http://image.clickear.top/20230613003409.png)

Compaction可以认为是，SMO的过程。在这个过程中，会称为性能瓶颈。

![](http://image.clickear.top/20230830170756.png)

### LSM的组成部分

#### 2.1 Memtable

MemTable 是 LSM-Tree 在内存中的数据结构，只用于保存最新的数据，按照 Key 有序地组织这些数据。  
LSM-Tree 没有规定用怎样的数据结构实现 MemTable，例如 HBase 使用跳表来保证内存中 Key 的有序性。  
存在内存中的数据会因为断电丢失，所以我们通常使用 WAL，即预写日志的方式来保证数据的可靠。

> WAL：预写日志，即事务的所有修改在提交之前要先写入 log 文件中

#### 2.2 Immutable MemTable

MemTable 达到一定大小后，会转化为 Immutable MemTable。Immutable MemTable 是将 MemTable 转为磁盘上的 SSTable 的一种中间状态。  
转化过程中写操作由新的 MemTable 处理，过程中不阻塞数据更新操作。

#### 2.3 SSTable

SSTable 是**有序键值对集合**，是 LSM 树在磁盘中的数据结构。它是一种持久化、有序且不可变的键值村存储结构

SSTable 内部包含一系列可配置大小的 Block 块。这些 Block 的 index 会被存储在 SSTable 的尾部，用于帮助快速查找特定的 Block。当一个 SSTable 被打开时，index 表会被加载到内存，然后根据 key 在内存 index 中进行一个二分查找，查到该 key 对应的磁盘的 offset 后，去磁盘把响应的块数据读取出来。  
![image.png](http://image.clickear.top/20230830171706.png)

当然，如果内存足够大，可以直接利用 MMAP 的技术把 SSTable 映射到内存中，提供更快的查找。

MemTable 达到一定大小会被 flush 到硬盘中变成 SSTable。在不同的 SSTable 中可能存在相同的 Key 记录。但这样会带来一些问题：

- **冗余存储**。对于某个 Key，除了最新的记录，其他记录都是冗余无用的。所以我们需要进行 Compact 操作（合并多个 SSTable），来清除冗余的记录。
- 读取时需要从最新的 SSTable 出发进行查询，最坏情况下药查询完所有的 SSTable。可以通过索引或布隆过滤器来优化查找速度。

### 3. LSM-Tree读写数据

#### 3.1 LSM-Tree写数据流程

LSM 树中，我们按照下面的步骤处理写数据请求。

1. 当收到写请求，先将数据记录在 WAL Log 中，用作故障恢复。
2. 将数据写入内存的 MemTable 中。为了有序，我们往往用跳表或红黑树实现。
    - 如果是删除，则做墓碑标记
    - 如果是更新，则新记录一条数据。
3. MemTable 达到一定大小后，在内存中冻结，成为不可变的 ImmuTable MemTable。同时也要生成新的 MemTable 来提供服务。
4. 内存中不可变的 MemTable 被 dump 到硬盘上的 SSTable 中，这也称为 **Minor Compaction。**注意 L0 层的 SSTable 是没有合并的，所以 key 在多个 SSTable 中往往会重叠、冗余。
5. 当每层 SSTable 超过一定大小，就会周期性的进行合并，这也称为 **Major Compaction**。这个阶段回清除掉荣誉的数据，防止浪费空间。由于 SSTable 都是有序的，可以使用归并排序进行高效合并。

#### 3.2 LSM-Tree读数据流程

1. 当收到读请求，现在内存中查询，查询到就返回。
2. 如果没有查询到，由内存到磁盘，在各级 SSTable 中依次下沉，直到得到结果。

###  LSM-Tree的Compact策略（LSM的重难点）

 控制 Compaction 的顺序和时间。常见的有 size-tiered 和 leveled compaction。LevelDB 便是支持后者而得名。前者比较简单粗暴，后者性能更好，也因此更为常见。leveled 策略相比 size-tiered 策略来说，每层内的 Key 是**有序、不重复**的。这样就很好地控制了冗余 Key 的量

#### size-tiered（限制每层数量）

**size-tiered策略，保证每层中每个 SSTable 的大小相近，同时限制每一层 SSTable 的数量**  
每层限制有 N 个 SSTable，每层数量达到 N 后，触发 Compact 操作来合并这些 SSTable，放入下一层成为更大的 SSTable  
当层数越来越大，单个 SSTable 的大小也会越来越大。该策略会导致空间放大比较严重。对每一层的 SSTable 来说，每个 key 的记录也可能存在多份。只有该层执行 Compact 操作才会消除这些冗余记录  
![image.png](http://image.clickear.top/20230830172220.png)

#### leveled策略(限制每层总大小)

**leveled 策略限制每一层总文件的大小。**  
** leveld 同样将每一层划分为大小相近的 SSTable。并保证在一层内全局有序。**这意味着与一个 Key 在每一层至多只有一条记录，不存在冗余记录  
![image.png](http://image.clickear.top/20230830172505.png)

### LSM的优化

+ **压缩**.例如，**前缀压缩**等。 SSTable可以进行压缩，而且不是压缩整个 SSTable。而是根据局部性原理将数据分组。每个分组分别压缩。这样读取数据的时候我们就不需要解压缩整个文件，而是解压缩部分 Group 即可读取。
+ **缓存** SSTable 除了进行 Compaction，其他情况下是不可变的。所以我们可以将一次扫描到的 Block 进行缓存，提高下一次检索的效率。
+ **索引/布隆过滤器** 正常情况下，一次读操作需要读取所有 SSTable，再将结果合并后返回。但是对某些 Key 而言，有些 SSTable 根本不包含对应数据。所以我们可以为每个 SSTable 添加布隆过滤器。来判断当前 SSTable 有没有我们需要的 Key。

- **合并** 合并本身肯定可以优化数据的组织情况，提高查询效率。但是也要注意查询是非常消耗 CPU 和磁盘 IO 的操作。一般我们选在业务量不大的凌晨等情况进行合并。

## 对比

|存储引擎|B-Tree|LSM-Tree|备注|
|---|---|---|---|
|优势|读取更快|写入更快||
|写放大|1. 数据和 WAL  <br>2. 更改数据时多次覆盖整个 Page|1. 数据和 WAL  <br>2. Compaction|SSD 不能过多擦除。因此 SSD 内部的固件中也多用日志结构来减少随机小写。|
|写吞吐|相对较低：  <br>1. 大量随机写。|相对较高：  <br>1. 较低的写放大（取决于数据和配置）  <br>2. 顺序写入。  <br>3. 更为紧凑。||
|压缩率|1. 存在较多内部碎片。|1. 更加紧凑，没有内部碎片。  <br>2. 压缩潜力更大（共享前缀）。|但紧缩不及时会造成 LSM-Tree 存在很多垃圾|
|后台流量|1. 更稳定可预测，不会受后台 compaction 突发流量影响。|1. 写吞吐过高，compaction 跟不上，会进一步加重读放大。  <br>2. 由于外存总带宽有限，compaction 会影响读写吞吐。  <br>3. 随着数据越来越多，compaction 对正常写影响越来越大。|RocksDB 写入太过快会引起 write stall，即限制写入，以期尽快 compaction 将数据下沉。|
|存储放大|1. 有些 Page 没有用满|1. 同一个 Key 存多遍||
|并发控制|1. 同一个 Key 只存在一个地方  <br>2. 树结构容易加范围锁。|同一个 Key 会存多遍，一般使用 MVCC 进行控制。|

## 资料

[简述LSM-Tree - pedro7 - 博客园](https://www.cnblogs.com/WangXianSCU/p/15939129.html)  
[DDIA 读书笔记（三）：B-Tree 和 LSM-Tree | 木鸟杂记](https://www.qtmuniao.com/2022/04/16/ddia-reading-chapter3-part1/)
