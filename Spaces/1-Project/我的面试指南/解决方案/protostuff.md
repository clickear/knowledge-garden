---
title: protostuff
date created: 2023-11-10
date modified: 2023-11-10
---

protostuff是一个基于protobuf实现的序列化方法，它较于protobuf最明显的好处是，在几乎不损耗性能的情况下做到了不用我们写.proto文件来实现序列化。

官方文档：[https://protostuff.github.io/docs/protostuff-runtime/](https://protostuff.github.io/docs/protostuff-runtime/)

> [!TIP] 注意事项：💡  
> 1、不能删除已有字段  
> 2、在不使用@tag注解的情况下，不能随意变更字段顺序，新增字段只能在末尾  
> 3、当使用@tag注解标识字段顺序时，其值在父类子类中要全局唯一  
> 4、字段的类型不能改变  
> 5、 不能加@depreted注解
