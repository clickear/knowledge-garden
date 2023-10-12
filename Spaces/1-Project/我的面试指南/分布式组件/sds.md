---
title: sds
date created: 2023-10-13
date modified: 2023-10-13
---

# sds

> [!TIP] 技巧💡  
> 简单动态字符串，

## 基本结构

先来看下基本的结构， 类似java中的String,需要有个长度标识len，当前字符串的大小，而不是以\0结尾来标识，但是为了兼容C，buf内容的最后，会默认加上\0.  
alloc，是总共分配内存大小。进行预分配，有效使用内存。  
flags: 主要用来表示类型，除了紧凑型，我暂时想不到为什么需要  
buf: 真正内容。  
![image.png](http://image.clickear.top/20231013020859.png)

+ **二进制安全**: SDS可以保存二进制数据，而不是以NULL结尾的字符串。这使得 Redis 可以存储非字符串的数据类型，例如二进制数据和经序列化的对象等
+ 预分配内存、惰性空间释放。减少了频繁的分配内存与释放。  
注意点:

## 紧凑型:

![sds.png](http://image.clickear.top/sds.png)

## encoding中raw和embstr的区别

![image.png](http://image.clickear.top/20231013023947.png)

embstr: 在分配内存的时候，是直接和sds分配的，而不是额外分配。  
![image.png](http://image.clickear.top/20231013023425.png)

## 资料

[Lion Long - 知乎](https://www.zhihu.com/people/long-xu-88-89/zvideos)
