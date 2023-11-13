---
title: 二、命令使用
date created: 2023-11-11
date modified: 2023-11-11
---

## 开发规范

## key名设计

### 1. key 名设计

- ### (1)【建议】: 可读性和可管理性
- ### (2)【建议】：简洁性

  保证语义的前提下，控制 key 的长度，当 key 较多时，内存占用也不容忽视，例如：  
      user:{uid}:friends:messages:{mid}简化为 u:{uid}:fr:m:{mid}。

- ### (3)【强制】：不要包含特殊字符

 反例：包含空格、换行、单双引号以及其他转义字符

### 2. value 设计

- ### (1)【强制】：拒绝 bigkey(防止网卡流量、慢查询)redis的key和value大小有限制：

redis的key和string类型value限制均为512MB。  
String类型：一个String类型的value最大可以存储512M  
List类型：list的元素个数最多为2^32-1个，也就是4294967295个。  
Set类型：元素个数最多为2^32-1个，也就是4294967295个。  
Hash类型：键值对个数最多为2^32-1个，也就是4294967295个。  
Sorted set类型：跟Set类型相似。

以上限制很宽泛，请参照以下规范设计:

         string 类型控制在 10KB 以内，hash、list、set、zset 元素个数不要超过 5000。 

         反例：一个包含 200 万个元素的 list。

        例如一个 200 万的 zset 设置 1 小时过期，会触发 del 操作，造成阻塞，而且该操作不会出现在慢查询中 (latency 可查))

- ### (2)【推荐】：选择适合的数据类型。

        例如：实体类型 (要合理控制和使用数据结构内存编码优化配置, 例如 ziplist，但也要注意节省内存和性能之间的平衡)

        反例：

    set user:1:name tom  
    set user:1:age 19  
    set user:1:favor football

      正例:

    hmset user:1 name tom age 19 favor football

- ### (3)【推荐】：选择合适的序列化选择

选择合适的[[序列化器]]和[[压缩算法]]，能减少value的体积

### 3.【强制】：控制 key 的生命周期，redis 不是垃圾桶。

      建议使用 expire 设置过期时间 (条件允许可以打散过期时间，防止集中过期)，不过期的数据重点关注 idletime。

### 4.【推荐】：设置合理的过期时间。

      如果大量key（尤其是大key）过期时间设置得过长就会常驻内存，内存满时触发key淘汰；另一方面如果将key的过期时间设得太短，可能导致缓存命中率过低并且内存闲置。

实际使用中，要根据业务场景设置合适的过期时间。

# 二、命令使用

## 1.【推荐】 O(N) 命令关注 N 的数量

例如 hgetall、lrange、smembers、zrange、sinter 等并非不能使用，但是需要明确 N 的值。

## 2.【推荐】：禁用命令

禁止线上使用 keys、flushall、flushdb、configset 等，通过 redis 的 rename 机制禁掉命令，或者使用 scan 的方式渐进式处理。

#### 3.【推荐】使用批量操作提高效率

```
原生命令：例如mget、mset。
```

但要注意控制一次批量操作的**元素个数**(例如500以内，实际也和元素字节数有关)。

## 参考文档

[阿里云Redis开发规范-阿里云开发者社区](https://developer.aliyun.com/article/531067)
