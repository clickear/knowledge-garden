---
date created: 2022-09-05
date modified: 2022-09-05
title: ParallelNew
---
> [!NOTE] 并行回收，  
>  和[[Parallel]]基本一致，能和CMS配合。
>  
>  追求降低用户停顿时间，适合交互式应用

- -XX:UseParNewGC手工指定ParNew收集器执行内存回收任务，它表示年轻代使用，不影响老年代
- -XX:ParallelGCThreads限制线程数量，默认开启和CPU数据相同的线程数
