---
date created: 2022-09-15
date modified: 2024-06-05
title: netty
tags: [todo/continue]
---

#todo/continue

按此方法重写梳理: ⭐ [19 | 链式 & 比较 & 环式学习法：怎么多维度提升技术能力？-大厂晋升指南-极客时间](https://time.geekbang.org/column/article/331463)

[BIO&NIO&AIO模型快速实战_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1JB4y1R7XB?p=1)

netty和java sdk的使用  
在java sdk nio的基础上，进行二次封装。

![](http://image.clickear.top/20220914154532.png)

连接过程:

1. bootstrap
2. ni event loop group 线程池

717 900万， qps 1000

NIO non-block IO非阻塞IO

1. netty,基于IO多路复用的事件驱动模型(select,epollo)的非阻塞IO框架.可定制化的**线程模型**.业务只要关注ChannelHandler的业务实现。
2. **[[零拷贝]]** 技术。除了操作系统级别的零拷贝技术外，Netty 提供了更多面向用户态的零拷贝技术，例如 Netty 在 I/O 读写时直接使用 DirectBuffer，从而避免了数据在堆内存和堆外内存之间的拷贝。
3. dubbo,rocketmq等都在使用  
线程模型， 主从reactor, boss(连接请求) worker(IO处理)  
![](http://image.clickear.top/20220914153527.png)

LengthFieldBasedFrameDecoder 不定长  
new IdeleStateHandle 心态检测  
自定义协议  
加密功能

[【史上最全】这绝对是目前B站最好的Netty面试题教程，Netty面试你看这个就够了！_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1JB4y1R7XB?spm_id_from=333.337.search-card.all.click)
