---
title: redis-对象共享
date created: 2023-11-08
date modified: 2023-11-08
---

> [!TIP] 技巧💡  
> 我们知道，redis的对象，使用的是[[redisObject]] 进行存储。在刚启动的时候，redis会创建一些整数的**共享对象池**来使用，而不是创建对象。节省内存。与Integer的缓存是一样的思想。
> 1. 创建整数对象范围: 0 ~ `OBJ_SHARED_INTEGERS`.OBJ_SHARED_INTEGERS可以配置
> 2. 共享对象的，[[redisObject]]中的refcout，初始化时，默认设置成最大值(旧版引用计数初始是0)。Interger.max

## 源码

### 服务启动时，创建共享池对象

```c
// server.c
void createSharedObjects(void) {
	int j;
	/* Shared command responses */
	shared.crlf = createObject(OBJ_STRING,sdsnew("\r\n"));
	shared.ok = createObject(OBJ_STRING,sdsnew("+OK\r\n"));
	// ....
	for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
		// 初始化对象
		shared.integers[j] =
		makeObjectShared(createObject(OBJ_STRING,(void*)(long)j));
		shared.integers[j]->encoding = OBJ_ENCODING_INT;
	}
}
```

```c 
robj *makeObjectShared(robj *o) {
	serverAssert(o->refcount == 1);
	// 新版中，引用数默认为最大值
	o->refcount = OBJ_SHARED_REFCOUNT;
	return o;
}
```

### 使用共享池对象限制条件

整数对象共享池与maxmemory和maxmemory-policy设置冲突，原因是：

1. LRU算法需要获取对象最后被访问时间， 以便淘汰最长未访问数据， 每个对象最后访问时间存储在redisObject对象的lru字段。 对象共享意味着多个引用共享同一个redisObject， 这时lru字段也会被共享， 导致无法获取每个对象的最后访问时间  
2. 如果没有设置maxmemory，那么服务器内存耗尽时都不会驱逐键，一般会报oom的错误，所以对象共享池能正常工作  
所以：当设置了maxmemory+maxmemory-policy（如volatile-lru，allkeys-lru,volatile-lfu,allkeys-lfu），对象共享池是禁用的

```c
//object.c
// 尝试对string对象进行编码，减少内存占用
robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;
    size_t len;

    // 非en
    if (!sdsEncodedObject(o)) return o;

	// 引用数>0，修改不安全，不编码
     if (o->refcount > 1) return o;

	 
    len = sdslen(s);
    // 如果字符串，能转换成long类型，则尝试转换
    if (len <= 20 && string2l(s,len,&value)) {
        /* This object is encodable as a long. Try to use a shared object.
         * Note that we avoid using shared integers when maxmemory is used
         * because every object needs to have a private LRU field for the LRU
         * algorithm to work well. */
        // 启用了 最大内存并且启用lfu,lru时，就不能使用整数共享对象。避免被回收
        if ((server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
            value >= 0 &&
            value < OBJ_SHARED_INTEGERS)
        {
            decrRefCount(o);
            incrRefCount(shared.integers[value]);
            return shared.integers[value];
        } else {
            if (o->encoding == OBJ_ENCODING_RAW) {
                sdsfree(o->ptr);
                o->encoding = OBJ_ENCODING_INT;
                o->ptr = (void*) value;
                return o;
            } else if (o->encoding == OBJ_ENCODING_EMBSTR) {
                decrRefCount(o);
                return createStringObjectFromLongLongForValue(value);
            }
        }
    }

    /* If the string is small and is still RAW encoded,
     * try the EMBSTR encoding which is more efficient.
     * In this representation the object and the SDS string are allocated
     * in the same chunk of memory to save space and cache misses. */
    // 转换成 embstr
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb;

        if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
        emb = createEmbeddedStringObject(s,sdslen(s));
        decrRefCount(o);
        return emb;
    }

    /* We can't encode the object...
     *
     * Do the last try, and at least optimize the SDS string inside
     * the string object to require little space, in case there
     * is more than 10% of free space at the end of the SDS string.
     *
     * We do that only for relatively large strings as this branch
     * is only entered if the length of the string is greater than
     * OBJ_ENCODING_EMBSTR_SIZE_LIMIT. */
    // 缩减空间大小
    trimStringObjectIfNeeded(o);

    /* Return the original object. */
    return o;
}
```

## 为什么只有整数对象共享池？

> [!TIP] 通用返回等信息对象也是有共享💡  
>  这个说明不太准确，其实在通用返回的错误信息对象，也是有共享的。参考共享对象创建过程

首先整数对象池复用的几率最大， 其次对象共享的一个关键操作就是判断相等性， Redis之所以只有整数对象池， 是因为整数比较算法时间复杂度为O（ 1） ， 只保留一万个整数为了防止对象池浪费。 如果是字符串判断相等性， 时间复杂度变为O（ n） ， 特别是长字符串更消耗性能（ 浮点数在Redis内部使用字符串存储） 。 对于更复杂的数据结构如hash、 list等， 相等性判断需要O（ n2） 。 对于单线程的Redis来说， 这样的开销显然不合理， 因此Redis只保留整数共享对象池
