---
title: redis-如何排查bigkey
date created: 2023-11-23
date modified: 2023-11-23
---

> [!TIP] 技巧💡
> 1. 官方工具查询带 --bigkeys 参数，./redis-cli --bigkeys
> 2. 定时任务跑

## –bigkeys选项(不建议用，会阻塞主线程)

```
./redis-cli  --bigkeys -i 0.1
// -i 控制扫描间隔
-------- summary -------
Sampled 32 keys in the keyspace!
Total key length in bytes is 184 (avg len 5.75)

//统计每种数据类型中元素个数最多的bigkey
Biggest   list found 'product1' has 8 items
Biggest   hash found 'dtemp' has 5 fields
Biggest string found 'page2' has 28 bytes
Biggest stream found 'mqstream' has 4 entries
Biggest    set found 'userid' has 5 members
Biggest   zset found 'device:temperature' has 6 members

//统计每种数据类型的总键值个数，占所有键值个数的比例，以及平均大小
4 lists with 15 items (12.50% of keys, avg size 3.75)
5 hashs with 14 fields (15.62% of keys, avg size 2.80)
10 strings with 68 bytes (31.25% of keys, avg size 6.80)
1 streams with 4 entries (03.12% of keys, avg size 4.00)
7 sets with 19 members (21.88% of keys, avg size 2.71)
5 zsets with 17 members (15.62% of keys, avg size 3.40)
```

### 缺点:

1. 无法获取topn
2. 对于集合类型来说，这个方法只统计集合元素个数的多少，而不是实际占用的内存量。但是，一个集合中的元素个数多，并不一定占用的内存就多。因为，有可能每个元素占用的内存很小，这样的话，即使元素个数有很多，总内存开销也不大。

## 自行开发程序

> [!TIP] 技巧💡  
>  .使用 SCAN 命令对数据库扫描，然后用 TYPE 命令获取返回的每一个 key 的类型。接下来，对于 String 类型，可以直接使用 STRLEN 命令获取字符串的长度，也就是占用的内存空间字节数  
>  对于集合类型，  
> 	 方案1: 集合数量 O(1） * 预估集合大小  
> 	 方案2: MEMORY USAGE获取键占用大小

方案设计:  
[[wiki:213984062]]  
[[wiki:24112290]]

## rdb dump快照，进行分析

## 资料

[22 第11～21讲课后思考题答案及常见问题答疑](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Redis%20%e6%a0%b8%e5%bf%83%e6%8a%80%e6%9c%af%e4%b8%8e%e5%ae%9e%e6%88%98/22%20%20%e7%ac%ac11%ef%bd%9e21%e8%ae%b2%e8%af%be%e5%90%8e%e6%80%9d%e8%80%83%e9%a2%98%e7%ad%94%e6%a1%88%e5%8f%8a%e5%b8%b8%e8%a7%81%e9%97%ae%e9%a2%98%e7%ad%94%e7%96%91.md)
