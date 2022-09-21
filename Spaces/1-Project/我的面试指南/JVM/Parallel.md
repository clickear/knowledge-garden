---
date created: 2022-09-05
date modified: 2022-09-05
title: Parallel
---

> [!NOTE] 并行回收，吞吐量优先  
>  并行是指，*多条*线程回收线程进行垃圾清理，清理需要暂停用户线程(即STW)  
>  追求 CPU 吞吐量，能够在较短时间内完成指定任务，因此适合没有交互的后台计算

- 通过参数 -XX:GCTimeRadio 设置垃圾回收时间占总 CPU 时间的百分比。
- 通过参数 -XX:MaxGCPauseMillis 设置垃圾处理过程最久停顿时间。
- 通过命令 -XX:+UseAdaptiveSizePolicy 开启自适应策略。我们只要设置好堆的大小和 MaxGCPauseMillis 或 GCTimeRadio，收集器会自动调整新生代的大小、Eden 和 Survivor 的比例、对象进入老年代的年龄，以最大程度上接近我们设置的 MaxGCPauseMillis 或 GCTimeRadio。
- XX:PreTenureSizeThreshold 大对象到底多大
- XX:MaxTenuringThreshold
- XX:+ParallelGCThreads 并行收集器的线程数，同样适用于CMS，一般设为和CPU核数相同
