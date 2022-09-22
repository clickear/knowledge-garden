---
date created: 2022-09-22
date modified: 2022-09-22
title: dubbo
---

## 简单版

1. 动态代理 --> 通过javassist来实现。也可以自定义SPI
2. 网络传输，netty建立长连接 (
	1. 序列化 obj-->byte[]
		1. hessian,hessian2
		2. pb,使用 proto 编译器，自动进行序列化和反序列化，速度非常快。压缩效果好。
		3. fst
		4. Kryo,
	2. dubbo协议，TCP黏包等问题)  

## 进阶版（带集群能力）:

1. 服务发现(注册中心)， 客户端，服务端启动后，注册到注册中心，dubbo使用zk。客户端会缓存一份，所以启动后注册中心挂了也没事。
2. 选择具体服务，cluster层将多个invoker封装成1个invoker
	1. dicretory, list 获取全部服务提供者信息
	2. 路由策略，router，过滤。 tag,script,condition
	3. 负载均衡，选择一个合适的。 轮询，权重，一致性hash，权重随机
3. 异常重试，容错机制
	1. failover，失败，自动重试到其它机器。（默认
	2. failfast,快速失败。一次调用失败就立即失败，常见于非幂等性的写操作
	3. failsafe，忽略异常，常用于不重要的接口调用，比如记录日志
	4. failback,失败了后台自动记录请求，然后定时重发
	5. forking, fork之后，调用多个
	6. broadcacst， 广播所有
4. 优雅开/停机。
5. skywalking链路追踪

## 服务治理

1. 服务降级。HelloServiceMock
2. 失败，超时重试。

## 源码解析

SPI