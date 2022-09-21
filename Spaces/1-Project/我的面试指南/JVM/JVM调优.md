---
date created: 2022-09-06
date modified: 2022-09-21
title: JVM调优
---

## ::我的理解

JVM调优，都是通过配置[[JVM参数]]来进行调优。调优之前，首先要先确定调优的目的。[[垃圾回收器性能指标]]

## ::调优步骤

1. 熟悉业务场景（没有最好的垃圾回收器，只有最合适的垃圾回收器）
    1. 响应时间、停顿时间 [[CMS垃圾收集器|CMS]], [[G1]],[[ZGC]] （需要给用户作响应）
    2. 吞吐量 = 用户时间 /( 用户时间 + GC时间) [[Parallel]]
2. 选择回收器组合
3. 计算内存需求（经验值 1.5G 16G）
4. 选定CPU（越高越好）
5. 设定年代大小、升级年龄
6. 设定日志参数
    1. Xloggc:/opt/xxx/logs/xxx-xxx-gc  
       -%t.log  
       -XX:+UseGCLogFileRotation  
       -XX:NumberOfGCLogFiles=5  
       -XX:GCLogFileSize=20M  
       -XX:+PrintGCDetails  
       -XX:+PrintGCDateStamps  
       -XX:+PrintGCCause
    2. 或者每天产生一个日志文件
7. 观察日志情况

## ::调优思路

这个问题在面试中很容易问到，抓住核心回答。

现在都是分代 GC，调优的思路就是尽量让对象在新生代就被回收，防止过多的对象晋升到老年代，减少大对象的分配。

**需要平衡分代的大小、垃圾回收的次数和停顿时间**。

需要对 GC 进行完整的监控，监控各年代占用大小、YGC 触发频率、Full GC 触发频率，对象分配速率等等。

然后根据实际情况进行调优。

比如进行了莫名其妙的 Full GC，有可能是某个第三方库调了 System.gc。

Full GC 频繁可能是 CMS GC 触发内存阈值过低，导致对象分配不过来。

还有对象年龄晋升的阈值、survivor 过小等等，具体情况还是得具体分析，反正核心是不变的

## ::调优案例

思路:

1. 有一个50万PV的资料类网站（从磁盘提取文档到内存）原服务器32位，1.5G 的堆，用户反馈网站比较缓慢，因此公司决定升级，新的服务器为64位，16G 的堆内存，结果用户反馈卡顿十分严重，反而比以前效率更低了
    1. 为什么原网站慢? 很多用户浏览数据，很多数据load到内存，内存不足，频繁GC，STW长，响应时间变慢
    2. 为什么会更卡顿？ 内存越大，FGC时间越长
    3. 咋办？ PS -> PN + CMS 或者 G1
2. 系统CPU经常100%，如何调优？(面试高频) CPU100%那么一定有线程在占用系统资源，
    1. 找出哪个进程cpu高（top）
    2. 该进程中的哪个线程cpu高（top -Hp）
    3. 导出该线程的堆栈 ([[jstack]])
    4. 查找哪个方法（栈帧）消耗时间 (jstack)jstat
    5. 工作线程占比高 | 垃圾回收线程占比高
3. 系统内存飙高，如何查找问题？（面试高频）
    1. 导出堆内存 ([[jmap]])
    2. 分析 (jhat jvisualvm mat jprofiler … )
4. 如何监控JVM
    1. jstat jvisualvm jprofiler arthas top…

### 分类

1. 硬件升级系统反而卡顿的问题？
	1. > 内存变大了，原来的垃圾回收器想做一次GC的话，时间反而变得更长了。所以不是说内存变大你的系统一定变得越快，还得选定特定的垃圾回收器才可以。你可以选择ParNew+CMS或者G1或者ZGC
2. 线程池不当运用产生OOM问题（见[[内存泄露]]） 不断的往List里加对象（实在太LOW）
3. smile jira问题 加内存 + 更换垃圾回收器 [[G1]]。
   > 	实际系统不断重启 解决问题 加内存 + 更换垃圾回收器 G1 真正问题在哪儿？不知道 JDK8 开始就可以使用G1了，而且相对来说已经很完善了，到了JDK9的时候，默认就是G1，JDK11或13的时候[[CMS垃圾收集器|CMS]]就已经被干掉了，完成了它的历史使命。 现实当中真的有一部分系统就是在不断的重启，比如游戏服务器，过一段时间就会出现“系统正在维护中，请20分钟后再重试”。比如地下城，以前也是这样的。选择凌晨的时候，集群中逐台重启，实在不行就来一次大重启。比如12306，每天晚上11点～第二天7点，是不能买票的，在维护
4. [[直接内存]]溢出问题（少见）
   > 《深入理解Java虚拟机》P59，使用Unsafe分配直接内存，或者使用NIO的问题。
5. 栈溢出问题
   > -Xss设定太小
6. 重写finalize引发频繁GC
   > 小米云，HBase同步系统，系统通过nginx访问超时报警，最后排查，C++程序员重写finalize引发频繁GC问题 为什么C++程序员会重写finalize？（new delete） finalize耗时比较长（200ms）
7. 如果有一个系统，内存一直消耗不超过10%，但是观察GC日志，发现FGC总是频繁产生，会是什么引起的？
   > System.gc() (这个比较Low)
8. Distuptor有个可以设置链的长度，如果过大，然后对象大，消费完不主动释放，会溢出.

## ::内存泄露示例,jstack 栈

```java
package com.mashibing.jvm.gc;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * 从数据库中读取信用数据，套用模型，并把结果进行记录和传输
 */

public class T15_FullGC_Problem01 {

    private static class CardInfo {
        BigDecimal price = new BigDecimal(0.0);
        String name = "张三";
        int age = 5;
        Date birthdate = new Date();

        public void m() {}
    }

    private static ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(50,
            new ThreadPoolExecutor.DiscardOldestPolicy());

    public static void main(String[] args) throws Exception {
        executor.setMaximumPoolSize(50);

        for (;;){
            modelFit();
            Thread.sleep(100);
        }
    }

    private static void modelFit(){
        List<CardInfo> taskList = getAllCardInfo();
        taskList.forEach(info -> {
            // do something
            executor.scheduleWithFixedDelay(() -> {
                //do sth with info
                info.m();

            }, 2, 3, TimeUnit.SECONDS);
        });
    }

    private static List<CardInfo> getAllCardInfo(){
        List<CardInfo> taskList = new ArrayList<>();

        for (int i = 0; i < 100; i++) {
            CardInfo ci = new CardInfo();
            taskList.add(ci);
        }

        return taskList;
    }
}
``` 

1. 代码  
2.java -Xms200M -Xmx200M -XX:+PrintGC com.mashibing.jvm.gc.T15_FullGC_Problem01
1. 一般是运维团队首先受到报警信息（CPU Memory）
2. top命令观察到问题：内存不断增长 CPU占用率居高不下
3. top -Hp 观察进程中的线程，哪个线程CPU和内存占比高
4. jps定位具体java进程 [[jstack]] 定位线程状况，重点关注：WAITING BLOCKED eg. waiting on <0x0000000088ca3310> (a java.lang.Object) 假如有一个进程中100个线程，很多线程都在waiting on xx ，一定要找到是哪个线程持有这把锁 怎么找？搜索jstack dump的信息，找xx ，看哪个线程持有这把锁RUNNABLE 作业：1：写一个死锁程序，用[[jstack]]观察 2 ：写一个程序，一个线程持有锁不释放，其他线程等待
5. 为什么阿里规范里规定，线程的名称（尤其是线程池）都要写有意义的名称 怎么样自定义线程池里的线程名称？（自定义ThreadFactory）

## ::堆 [[jmap]]

[[jmap]]

jmap - histo 4655 | head -20，查找有多少对象产生

## ::常见问题

1.生产环境中，倾向于将最大堆内存和最小堆内存设置为：（为什么？）  
总结：*避免每次垃圾回收完成后JVM重新分配内存。*

> A: 相同 B：不同  
  A，好处是：  
   1） 避免JVM在运行过程中向操作系统申请内存  
   2）延后启动后首次GC的发生时机  
   3）减少启动初期的GC次数

2. 什么是响应时间优先？  
      注重的是垃圾回收时STW的时间最短。
3. 什么是吞吐量优先？  
      吞吐量是指应用程序线程用时占程序总用时的比例，也就是说尽量躲让用户程序去执行。
4. ParNew和PS的区别是什么？  
      都是年轻代多线程收集器。  
      ParNew 回收器是通过控制 垃圾回收 的 线程数 来进行参数调整，而 Parallel Scavenge 回收器更关心的是`程序运行的吞吐量`。即一段时间内，用户代码 运行时间占 总运行时间 的百分比。
5. ParNew和ParallelOld的区别是什么？（年代不同，算法不同） 、  
      前者是年轻代收集器，后者是老年代收集器，然后解释两者。
6. 长时间计算的场景应该选择：吞吐量优先的收集器和策略。
    
7. 大规模电商网站应该选择：停顿时间少（即响应时间快）的收集器和策略。
    
8. JDK1.7 1.8 1.9的默认垃圾回收器是什么？如何查看？  
      jdk1.7 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）；  
      jdk1.8 默认垃圾收集器Parallel Scavenge（新生代）+Parallel Old（老年代）；  
      jdk1.9 默认垃圾收集器G1。  
      `java -XX:+PrintCommandLineFlags -version`命令可以查看使用的垃圾回收器。
9. 所谓调优，到底是在调什么？  
      本人认为，是根据业务需要，在吞吐量和响应时间之间做出选择。
10. 如果采用PS + ParrallelOld组合，怎么做才能让系统基本不产生FGC  
      应该和下个问题的答案是有相通之处的。
11. 如果采用ParNew + CMS组合，怎样做才能够让系统基本不产生FGC  
      1）加大JVM内存  
      2）加大Young（年轻代）的比例  
      3）提高Y-O（最大值是15）的年龄  
      4）提高S（survivor）区比例  
      5）避免代码内存泄漏
    
12. 如果G1产生FGC，你应该做什么？  
      1）扩内存  
      2）提高CPU性能（回收的快，业务逻辑产生对象的速度固定，垃圾回收越快，内存空间越大）  
      3）`降低MixedGC触发的阈值，让MixedGC提早发生（默认是45%）`。具体的参数是：

```java
-XX:InitiatingHeapOccupancyPercent=45
```

13. 问：生产环境中能够随随便便的dump吗？  
    小堆影响不大，大堆会有服务暂停或卡顿（加live可以缓解），dump前会有FGC  
14. 问：常见的OOM问题有哪些？  
    栈 堆 MethodArea 直接内  
15. 如果JVM进程静悄悄退出怎么办？  
    1. JVM自身OOM导致  
        1. heap dump on oom，这种最容易解决  
    2. JVM自身故障  
        1. XX:ErrorFile=/var/log/hs_err_pid pid.log 超级复杂的文件 包括：crash线程信息 safepoint信息 锁信息 native code cache , 编译事件, gc相关记录 jvm内存映射 等等  
    3. 被Linux OOM killer杀死  
        1. 日志位于/var/log/messages  
        2. egrep -i 'killed process' /var/log/messages  
    4. 硬件或内核问题  
        1. dmesg | grep java  
4. **如何排查直接内存**？  
    1. NMT打开 -- -XX:NativeMemoryTracking=detail  
    2. perf工具  
    3. gperftools  
5. 有哪些常用的日志分析工具？  
    1. gceasy  
6. CPU暴增如何排查？  
    1. top -Hp jstack  
    2. [[arthas]] - dashboard thread thread XXXX  
    3. 两种情况：1：业务线程 2：GC线程 - GC日志  
7. 死锁如何排查？  
    1. jstack 观察线程情况  
    2. [[arthas]] - thread -b

## ::优秀调优经历blog

### YGC时间变长 --> 长生命周期的对象越来越多，导致标注和复制过程的耗时增加

YGC 时间变长

1. 猜测， 长生命周期的对象越来越多，导致标注和复制过程的耗时增加
2. 局部变量来说，在每次YGC后就能够马上被回收了。所以不会是局部变量
3. 猜测，全局变量或者类静态变量上。这些变量不会马上被回收。也不会马上到[[老年代]]
4. [[jmap]],查看有哪些大对象，进行猜测。后发现是静态变量一直添加，未去重
5. 其它思路:
   1. 对存活对象标注时间过长：**比如重载了Object类的Finalize方法，导致标注Final Reference耗时过长；或者String.intern方法使用不当，导致YGC扫描StringTable时间过长**。
   2. 长周期对象积累过多：比如本地缓存使用不当，积累了太多存活对象；或者锁竞争严重导致线程阻塞，局部变量的生命周期变长。

[YGC问题排查，又让我涨姿势了！ | HeapDump性能社区](https://heapdump.cn/article/1661497)

### 元空间内存泄露，[[直接内存]]溢出

内存上涨，宕机

1. 内存使用，超过了堆内存上限。猜测是[[直接内存]]溢出
2. 使用jcmd来查看堆外内存。需要jvm参数设置支持跟踪堆外内存
3. 启动参数新增 -verbose:class，发现fastjson有个类频繁创建
4. fastjson序列化，用使用asm创建动态代理类，导致动态代理类激增
  > 每个SerializeConfig实例若序列化相同的类, 都会找到之前生成的该类的代理类, 来进行序列化. 们的服务在每次接口被调用时, 都实例化一个ParseConfig对象来配置Fastjson反序列的设置, 而未禁用ASM代理的情况下, 由于每次调用ParseConfig都是一个新的实例, 因此永远也检查不到已经创建的代理类, 所以Fastjson便不断的创建新的代理类, 并加载到metaspace中, 最终导致metaspace不断扩张, 将机器的内存耗尽
   
   ```java
/**
   * 返回Json字符串.驼峰转_
 * @param bean 实体类.
 */
public static String buildData(Object bean) {
    try {
        SerializeConfig CONFIG = new SerializeConfig();
        CONFIG.propertyNamingStrategy = PropertyNamingStrategy.SnakeCase;
        return jsonString = JSON.toJSONString(bean, CONFIG);
    } catch (Exception e) {
        return null;
    }
}
```
   

[一次完整的JVM堆外内存泄漏java故障排查记录 - 知乎](https://zhuanlan.zhihu.com/p/432258798)

## :: 大厂真题

1. [[对象的创建过程]]
2. [[指令重排]]
3. 对象在内存中的存储布局?(对象和数组的存储不同).[[对象存储布局]]
4. [[指针压缩]]为什么超过32G就失效。
