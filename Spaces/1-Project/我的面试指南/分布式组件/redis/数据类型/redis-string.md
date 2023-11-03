---
title: redis-string
date created: 2023-11-02
date modified: 2023-11-02
---

## String

最简单的类型，可以存储 数字、字符串、序列化后的json对象、二进制对象等。value值不能大于512M.

> [!TIP] 技巧💡  
> 最常用的使用场景，
> 1. 缓存。可以缓存json对象等。
> 2. 分布式锁，使用setnx
> 3. counter,计算器
> 4. 限流器

我们常说的[[分布式锁]]

## 内部编码

[[sds]]

## 使用场景

#### 缓存

涉及到[[缓存和数据库一致性]]的问题。 如何缓存json对象时，应该要注意序列化器。为减少内存的占用，一般会转换成 byte[]数组，有些还会使用 snappy 进行压缩。在使用时，在进行解压

#### [[分布式锁]]

![image.png](http://image.clickear.top/20231008232819.png)

在6.12版本之后，才支持设置过期失效，保证原子性。  
在6.12之前，我们常说的setnx命令，是不支持设置过期时间的。所以需要加lua脚本保证原子性。

`SET lock_key unique_value NX PX 10000`

- lock_key 就是 key 键；
- unique_value 是客户端生成的唯一的标识；**释放时，需要判断，是否是同一个客户端**
- NX 代表只在 lock_key 不存在时，才对 lock_key 进行设置操作；
- PX 10000 表示设置 lock_key 的过期时间为 10s，这是**为了避免客户端发生异常而无法释放锁**。

PX --> pexpire  
ex --> expire  
exat --> expireat

#### counter

如果string是数字类型时，还支持自增操作。是线程安全的。类型java种的[[AtomicLong]].

```
incr key ==> incrby key 1  
incrby key count  
describe
```
