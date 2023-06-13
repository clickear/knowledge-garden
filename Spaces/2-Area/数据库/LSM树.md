---
title: LSM树
date created: 2023-06-13
date modified: 2023-06-13
---
 > [!TIP] Log-structured Merge tree💡  
>  SSD，没有寻道时间，减少擦除损坏

LSM树，在 1（读写性能）的性能不是特别好。主要是通过compaction来操作整理LSM树的结构。

![image.png](http://image.clickear.top/20230613003409.png)

Compaction可以认为是，SMO的过程。在这个过程中，会称为性能瓶颈。
