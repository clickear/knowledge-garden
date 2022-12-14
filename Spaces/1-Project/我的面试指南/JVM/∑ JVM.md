---
date created: 2022-09-01
date modified: 2022-09-02
title: ∑ JVM
---

## ::我的理解

> 什么是[[JAVA虚拟机]]，顾名思义，就是模拟通用的计算机,来进行执行代码。也就是说，需要每个平台都有虚拟机程序，在虚拟机程序上屏蔽操作系统的差异。然后执行自己的一套代码。想[[C语言]] 这种，都是通过每个平台的[[编译器]]上编译成每个平台可执行的二进制代码来执行。而java则是通过编译成 [[字节码]],来达到一次编译，到处执行。因为[[JAVA虚拟机]]执行的就是特定代码，即编译后的[[字节码]]

## ::架构图

![[JVM架构图]]

先经过[[前端编译]]，编译成class文件。然后通过[[类加载子系统]]加载到内存中，将方法等放到相应的方法区，常量区等。方便后续执行代码使用到。后通过JVM执行引擎执行代码。没调用一个方法，需要将[[栈帧]]压栈，然后结合堆区，方法区，常量区和程序计数器进行配合执行代码。JVM执行引擎，会收集runtime信息，判断是否为[[热点代码]],必要时进行编译成机器码。  
这里涉及到JVM常用的一个问题，就是堆区垃圾的释放问题。在C++中，是由代码进行手动释放的。JVM是自动释放垃圾的。这就涉及到[[垃圾回收]]

## 文献

[GitHub - doocs/jvm: 🤗 JVM 底层原理最全知识总结](https://github.com/doocs/jvm)
