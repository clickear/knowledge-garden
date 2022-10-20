---
date created: 2022-09-26
date modified: 2022-09-26
title: 推荐使用 LongAdder 对象，比 AtomicLong 性能更好（减少乐观锁的重试次数）
---

> [!NOTE] 笔记
> 1. AtomicLong做累加的时候实际上就是多个线程操作同一个目标资源。在高并发时，只有一个线程是执行成功的，其他的线程都会失败，不断自旋（重试），自旋会成为瓶颈。
> 2. LongAdder，把要操作的目标资源「分散」到数组Cell中。每个线程对自己的 Cell 变量的 value 进行原子操作，大大降低了失败的次数
