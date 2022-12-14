---
date created: 2022-09-02
date modified: 2022-11-23
title: § JVM目录
---

+ [[JVM架构图]]
+ JVM作用
	+ [[JAVA代码是如何运行的]]
	+ [[JAVA编译]]
		+ [[解释器与编译器]]
		+ [[字节码]]
+ [[类加载子系统]]
	+ 类加载过程
		+ [[加载]]
		+ [[校验]]
		+ [[准备]]
		+ [[解析]]
		+ [[初始化]]
	+ 类加载时机（按需加载，在用到时才加载）
+ [[运行时数据区]] （JVM内存结构）
	+ [[虚拟机栈]]
	+ [[本地方法栈]]
	+ [[堆]]
		+ [[新生代]]
		+ [[老年代]]
	+ [[元数据区]]
	+ [[程序计数器]]
+ [[垃圾回收]]
	+ 什么是垃圾？
		+ [[引用计数法]]
		+ [[可达性分析]]
	+ 引用，
		+ [[强引用]]，GC无法回收
		+ [[软引用]]，GC后内存还不够，可回收。主要用于内存
		+ [[弱引用]]，GC后回收，不管内存够不够
		+ [[虚引用]], 无法获取真实引用对象，GC后，触发GC后，要做的一些事情的机制
	+ [[垃圾回收算法]]
		+ [[标记清除法]]
		+ [[标记复制法]]
		+ [[标记整理法]]
		+ [[分代收集算法]] --根据分代，来进行使用回收算法
		+ [[增量收集算法]]
	+ [[垃圾收集器]]
		+ [[垃圾回收器性能指标]]
		+ [[Serial]] 串行回收
		+ [[Parallel]] 并行回收，吞吐量优先
		+ [[ParallelNew]] 可以结合CMS使用
		+ [[CMS垃圾收集器|CMS]],承上启下，垃圾收集线程和用户线程可同时运行
			+ [[三色标记法]]
			+ [[读写屏障|写屏障]] + [[增量更新]]，写时如果发现，黑色引用了白色，直接将这个引用记录下来。
		+ [[G1]]
			+ [[读写屏障|写屏障]] + [[SATB]]
		+ [[ZGC]]
			+ [[读写屏障|读屏障]] + [[颜色指针]]。读的时候，如果不对，要校对
+ [[JVM调优]]
	+ [[JVM参数]]

	
