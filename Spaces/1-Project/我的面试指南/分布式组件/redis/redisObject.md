---
title: redisObject
date created: 2023-10-13
date modified: 2023-11-08
---
> [!TIP] 技巧💡  
> [[redisObject]]总共占用16byte

```
typedef struct redisObject {
    unsigned type:4; // 4 bit
    unsigned encoding:4; // 4 bit
    unsigned lru:LRU_BITS; // 3 个字节
    int refcount; // 4 个字节
    void *ptr; // 8 个字节
} robj;

```

- type：对象的数据类型，例如：string、list、hash 等，占用 4 bits 也就是半个字符的大小；
- encoding：对象数据编码，占用 4 bits；
- lru：记录对象的 LRU(Least Recently Used 的缩写，即最近最少使用)信息，内存回收时会用到此属性，占用 24 bits(3 字节)；
- refcount：引用计数器，占用 32 bits(4 字节)；
- *ptr：对象指针用于指向具体的内容，占用 64 bits(8 字节)。

### lru字段

**在 [[redis-lru]]算法中**，Redis 对象头的 24 bits 的 lru 字段是用来记录 key 的访问时间戳，因此在 LRU 模式下，Redis可以根据对象头中的 lru 字段记录的值，来比较最后一次 key 的访问时间长，从而淘汰最久未被使用的 key。

**在 [[redis-lfu]]算法中**，Redis对象头的 24 bits 的 lru 字段被分成两段来存储，高 16bit 存储 ldt(Last Decrement Time)，低 8bit 存储 logc(Logistic Counter)。

![](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E8%BF%87%E6%9C%9F%E7%AD%96%E7%95%A5/lru%E5%AD%97%E6%AE%B5.png)

- ldt 是用来记录 key 的访问时间戳；
- logc 是用来记录 key 的访问频次，它的值越小表示使用频率越低，越容易淘汰，每个新加入的 key 的logc 初始值为 5。

注意，logc 并不是单纯的访问次数，而是访问频次（访问频率），因为 **logc 会随时间推移而衰减的**

## refcount [[引用计数法]]

> [!TIP] 技巧💡
> 1. 基于此字段，进行[[redis-对象共享]] 。  
> 2. 当直到引用计数为零时才会真正地释放该对象占用的内存。

## lru和refcout如何配合，进行内存回收

Redis 的内存回收是通过使用引用计数和惰性删除技术实现的. 具体来说, 当一个键被删除时, Redis 只是将该键对应的对象标记为已删除, 并将键对象从 Redis 数据库中删除. 如果该键对应的对象被其他键对象引用, Redis 会将该对象的引用计数[[redisObject]]中的refcout减一, 直到引用计数为零时才会真正地释放该对象占用的内存.
