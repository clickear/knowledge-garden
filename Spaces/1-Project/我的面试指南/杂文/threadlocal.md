---
date created: 2022-09-26
date modified: 2023-01-19
title: threadlocal
---

![[img_threadlocal]]

## threadlocal

在thread中，存在一个 threadlocalmap的对象。threadlocalMap是一个数组的实现，有多个entry,entry的key值，是一个弱引用，指向的是[[threadlocal]],而value就是实际存的值。  
思考:

1. 为什么需要虚引用？方便在外部，将threadlocal设置为null时，key值自动回收的case。
2. 为什么存在内存泄露的可能性？因为如果entry的key断开了连接，是null值。那边可能存在threadRef --> thread --> threadMap --> entry --> value的强引用，只是这个endtry的key是null而已。那就造成了gc无法进行回收。
	1. 解决方案。 threadlocal在get，set，remove都会将key为null的进行回收。最好就是在结束后，调用remove方法。

### ThreadLocalMap中的key，为什么是ThreadLocal，而不是Thread.

1. 一个线程是可以拥有多个私有变量的嘛, 那key如果是当前线程的话, 意味着还点做点「手脚」来唯一标识set进去的value。
2. 并发量足够大时, 意味着所有的线程都去操作同一个Map, Map体积有可能会膨胀, 导致访问性能的下降
3. 这个Map维护着所有的线程的私有变量, 意味着你不知道什么时候可以「销毁」（Map的生命周期问题）

### ThreadLocal内存泄露问题

> 内存泄露: 你申请完内存后, 你用完了但没有释放掉, 你自己没法用, 系统又没法回收.

ThreadLocal内存泄露指的是: ThreadLocal被回收了, ThreadLocalMap Entry的key没有了指向。但Entry仍然有ThreadRef->Thread->ThreadLoalMap-> Entry value-> Object 这条引用一直存在导致内存泄露，从中可看**ThreadLocalMap是依附在Thread上的**, 只要Thread销毁, 那ThreadLocalMap也会销毁出。

可能存在泄露的可能性:

1. 假如每个key都**强引用指向threadlocal**，也就是上图虚线那里是个强引用，那么这个threadlocal就会因为和entry存在强引用无法被回收！造成内存泄漏 ，除非线程结束，线程被回收了，map也跟着回收。
2. 当把threadlocal实例置为null以后,没有任何强引用指向threadlocal实例,所以threadlocal将会被gc回收.map里面的value却没有被回收.而**这块value永远不会被访问到**了. 所以存在着内存泄露。

<aside> 💡 Java为了最小化减少内存泄露的可能性和影响，在ThreadLocal的get,set的时候都会清除线程Map里所有key为null的value

</aside>

内存泄露条件（同时满足）：

1. ThreadLocal被回收，如设置成null
2. 线程被复用
3. 线程复用后不再调用ThreadLocal的set/get/remove方法

如何避免:

1. 使用完后，用remove方法

### 为什么要将ThreadLocalMap的key设置为[[弱引用]]呢? 强引用不香吗?

外界是通过ThreadLocal来对ThreadLocalMap进行操作的, 假设外界使用ThreadLocal的对象被置null了, 那ThreadLocalMap的强引用指向ThreadLocal也毫无意义。

![[img_threadlocal]]  
Thread在创建的时候, 会有栈引用指向Thread对象, Thread对象内部维护了ThreadLocalMap引用，而ThreadLocalMap的Key是ThreadLocal, value是传入的Object

1):ThreadLocalRef->ThreadLocal(强引用)

2):ThreadLocalMap Entry key ->ThreadLocal(弱引用)

### 使用场景

+ 用户上下文数据
+ MDC，存储。traceId等

## 如果自己实现的思路？

大概率，会优先考虑

```java
public class ThreadLocal<T> {

    private Map<Thread, T> threadValMap = Collections.synchronizedMap(new WeakHashMap<>());
    
    public void set(T value) {
        threadValMap.put(Thread.currentThread(), value);
    }

    public T get() {
        return threadValMap.get(Thread.currentThread());
    }

    // remove....
}
``` 

与threadLocal实现的差异:

1. key值不同，一个是threadlocal，一个是thread。 采用线程池进行线程复用时，那么Thread将一直保持引用，不能被GC掉，可能导致内存泄露。  
**ThreadLocal**倒是也有类似的问题，因为大多数情况我们都会使用**static**引用ThreadLocal，一直保持强引用。好在ThreadLocal提供了**remove**方法，只需开发者注意用完后调用即可
1. 需要加锁。而 **ThreadLocal**是**线程封闭**的。即不用加锁。  
   因为，如果是我们自己的实现，是放在一个Map中的。会存在多线程访问带来的线程安全问题。而ThreadLocal,是通过在thread中添加了一个threadlocalmap变量。只针对**自己当前线程进行**操作。不存在线程安全问题。

## [[FastThreadLocal]]为什么性能更高

解决[[hash冲突]]

1. 线性探测法(ThreadLocal)
2. 链表 + 红黑树(HashMap)
3. 数组法（FastThreadLocal）

### ThreadLocal使用的是线性探测的方式解决hash冲突

![](http://image.clickear.top/20211115152232.png)  
如果没有找到空闲的slot，就不断往后尝试，直到找到一个空闲的位置，插入entry，这种方式在经常遇到hash冲突时，影响效率。

存储或查询时会先计算ThreadLocal的哈希值模ThreaedMap的length-1的位置, 如果有占用寻找next值, 如果next占用继续循环直到找到未占用, 数据量大会增加查询时间, 并没有HashMap的存储查询算法优化的好

### FastThreadLocal使用数组避免冲突。

![[img_fastthreadlocal]]

```java
public class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}

public class FastThreadLocalThread extends Thread {
    private InternalThreadLocalMap threadLocalMap;
}
```

![](http://image.clickear.top/20211115152541.png)  
每一个FastThreadLocal实例创建时，分配一个下标index；分配index使用AtomicInteger实现，每个FastThreadLocal都能获取到一个不重复的下标。

当调用`ftl.get()`方法获取值时，直接从数组获取返回，如`return array[index]`

## 资料

[Java中的ThreadLocal通常是在什么情况下使用的？ - 知乎](https://www.zhihu.com/question/21709953/answer/2488516865?utm_campaign=shareopn&utm_medium=social&utm_oi=539749754213535744&utm_psn=1556913540828770305&utm_source=wechat_session)  
[ThreadLocal的设计优点在哪👀 - 掘金](https://juejin.cn/post/6844904062991892494)  
[Site Unreachable](https://jishuin.proginn.com/p/763bfbd697af)
