---
date created: 2022-09-19
date modified: 2022-09-19
title: 资料
---

> [!TIP] 技巧💡
> 1. 基于事件模式的IO模型。非阻塞IO
> 2. 几乎所有的网络连接都会经过  
>    accept: 连接  
>    read: 读取请求内容  
>    decode: 解码  
>    compute: 业务处理  
>    encode: 编码  
>    send: 发送，回复内容

## 发展史

> [!INFO] 提示  
> 1. 单线程,一般阻塞->多线程,一般阻塞（一条连接一线程）->线程池(减少线程创建销毁开销)->reactor(更小粒度的线程)  
> 2. 所谓更小的粒度的线程是指,传统的多线程是一个连接一个线程,粒度太大,比如可以把一个连接继续细分成三个步骤:read,process,send三个步骤,每个步骤占一个线程,处理完后交给主线程调度,进入下一个处理模块

### BIO

![](http://image.clickear.top/20220919151948.png)  
这种模型由于**IO在阻塞时会一直等待**，因此在用户负载增加时，性能下降的非常快。

#### BIO + 单线程

```java
while(true){  
socket = accept();  
handle(socket)  
}
```

> 服务器用一个while循环，不断监听端口是否有新的套接字连接，如果有，那么就调用一个处理函数处理。最大问题是无法并发，效率太低，如果当前的请求没有处理完，那么后面的请求只能被阻塞，服务器的吞吐量太低。

### BIO + 多线程 （经典的: connection per thread）

```java
while(true){  
socket = accept();  
new thread(socket);  
}
```

> 资源要求太高，系统中创建线程是需要比较高的系统资源的，如果连接数太高，系统无法承受，而且，线程的反复创建-销毁也需要代价。（就算用线程池，也是治标不治本。）

### reactor模式(基于事件驱动的非阻塞IO)

**采用基于事件驱动的设计，当有事件触发时，才会调用处理器进行数据处理**

在[[netty]]中，reactor是基于epollo或者select 进行多路复用。

#todo/continue epollo模型

#### 单reactor模式

连接，IO事件等都在一个线程。  
![](http://image.clickear.top/20220919152128.png)

#### 多线程reactor模式（**使用多线程处理业务逻辑**）

![](http://image.clickear.top/20220919152230.png)

将decode、compute、encode等处理器的执行放入线程池，多线程进行业务处理。但Reactor仍为单个线程。

#### 主从reactor模式 (对于多个CPU的机器，为充分利用系统资源，将Reactor拆分为两部分)

![](http://image.clickear.top/20220919152446.png)

mainReactor负责监听连接，accept连接给subReactor处理，为什么要单独分一个Reactor来处理监听呢？因为像TCP这样需要经过3次握手才能建立连接，这个建立连接的过程也是要耗时间和资源的，单独分一个Reactor来处理，可以提高性能。

#### [[rocketmq]]中的1+N+M1+M2

> [!NOTE] 笔记
> 1. 1个Acceptor
> 2. N（3）个处理IO事件
> 3. M1(8)个线程 编解码器
> 4. M2, 业务线程，即compute步骤

1. **一个** Reactor 主线程（eventLoopGroupBoss）负责监听 **TCP网络连接请求**，
2. 建立好连接后丢给Reactor 线程池（eventLoopGroupSelector，即为上面的“N”，源码中默认设置为3），它负责将建立好连接的socket 注册到 selector上去（RocketMQ的源码中会自动根据OS的类型选择NIO和Epoll，也可以通过参数配置），然后监听真正的网络数据。
3. 拿到网络数据后，再丢给Worker线程池（defaultEventExecutorGroup，即为上面的“M1”，源码中默认设置为8）。为了更为高效的处理RPC的网络请求，这里的Worker线程池是**专门用于处理Netty网络通信相关的（包括编码/解码、空闲链接管理、网络连接管理以及网络请求处理）**。而
4. 处理业务操作放在业务线程池中执行，即为"M2"，根据 RomotingCommand 的业务请求码code去processorTable这个本地缓存变量中找到对应的 processor，然后封装成task任务后，提交给对应的业务processor处理线程池来执行（sendMessageExecutor，以发送消息为例，即为上面的 “M2”）。  

-----------------------------------

# 资料

+ rocketmq中的netty使用: [29 从 RocketMQ 学 Netty 网络编程技巧.md](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/RocketMQ%20%E5%AE%9E%E6%88%98%E4%B8%8E%E8%BF%9B%E9%98%B6%EF%BC%88%E5%AE%8C%EF%BC%89/29%20%E4%BB%8E%20RocketMQ%20%E5%AD%A6%20Netty%20%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B%E6%8A%80%E5%B7%A7.md)
+ reactor发展史: [高性能IO之Reactor模式 - 时间朋友 - 博客园](https://www.cnblogs.com/doit8791/p/7461479.html)
+ rocketMQ线程模式: [分布式消息队列 RocketMQ 源码分析 —— RPC 通信（二)](https://blog.51cto.com/u_15310381/3233658)
