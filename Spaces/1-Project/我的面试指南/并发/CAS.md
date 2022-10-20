---
date created: 2022-09-26
date modified: 2022-09-26
title: CAS
---

> [!NOTE] 笔记  
>  经常会跟自旋挂钩，因为 CAS 有三个操作数：当前值A、内存值V、要修改的新值B。  
> 1. 假设 当前值A 跟 内存值V 相等，那就将 内存值V 改成B。  
> 2. 假设 当前值A 跟 内存值V 不相等，要么就重试，要么就放弃更新。

CAS用在很多地方，比如[[synchronized]]在锁升级时，会进行CAS进行变更。


## ABA问题
java也提供了AtomicStampedReference类供我们用，说白了就是加了个版本，比对的就是内存值+版本是否一致