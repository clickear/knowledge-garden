---
title: redisObject
date created: 2023-10-13
date modified: 2023-10-13
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
