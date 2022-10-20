---
date created: 2022-09-26
date modified: 2022-09-26
title: 资料
---

## 为什么说synchronized是重量级锁？

我们知道，cpu会运行在用户态和内核态。如[[零拷贝]]就是为了减少在用户态的运行。[[synchronized]]效率低， 是因为synchronized是跑在jvm上，需要对操作系统进行申请资源。  
在java中很多实现中, 很多都是轻量级锁, 比如JUC中的CAS. 所谓的轻量级锁和重量级锁的区别是什么呢? 轻量级锁都是在**用户态直接完成**, 不用惊动操作系统, 而重量级锁需要**向操作系统申请**. 在现在synchronized内部的执行过程中, 他会首先使用轻量级锁, 在用户态中完成, 如果完成不了才会去申请重量级锁, 即内核态的锁, 这就是synchronized的**升级**过程

> 跑在jvm上，会有资源损坏。比如dubbo的[[直接内存|堆外内存]]实现，就是为了减少jvm到c的heap的损坏。

## 锁的对象是什么？

> [!TIP] 技巧💡  
> 本质上，是给某个对象加了锁。
>
1. 修饰实例方法，对当前实例对象this加锁
2. 修饰静态方法，对当前类的Class对象加锁。Class对象，在jvm中也是一个对象。
3. 修饰代码块，指定加锁对象，对给定对象加锁

## [[synchronized]]的实现原理(重量级锁的原理)

在[[对象存储布局]]中可知，一个对象的内存布局方式。  
![](http://image.clickear.top/20220926104914.png)

重量级锁也就是通常说synchronized的对象锁，锁标识位为10，其中指针指向的是monitor对象（也称为管程或监视器锁）的起始地址。每个对象都存在着一个 monitor 与之关联。在Java虚拟机(HotSpot)中，monitor是由ObjectMonitor实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的），省略部分属性

```java

ObjectMonitor() {  
    _count = 0; //记录数  
    _recursions = 0; //锁的重入次数  
    _owner = NULL; //指向持有ObjectMonitor对象的线程  
    _WaitSet = NULL; //调用wait后，线程会被加入到_WaitSet  
    _EntryList = NULL ; //等待获取锁的线程，会被加入到该列表  
}
```

> [!NOTE] 笔记
> 1. 可以这么理解，如果是重量级锁，每个对象头指向的是监视器的指针，这个monitor每个对象都有一个与之关联。
> 2. 每个monitor，包含了  
> 	  同步队列(entryList)，即是都需要竞争抢资源的线程列表。  
> 	  当前线程，owner  
> 	  等待队列，即调用wait之后，会被放到等待队列。在被调用notify或者notifyAll时，会移动到同步队列中，重新竞争。

![](http://image.clickear.top/20220926110229.png)

## 锁升级

![](http://image.clickear.top/20220926115625.png)

> [!NOTE] 笔记  
> synchronized锁升级过程总结：**一句话，就是先自旋，不行再阻塞**。
>
> 实际上是把之前的悲观锁(重量级锁)变成在一定条件下使用偏向锁以及使用轻量级(自旋锁CAS)的形式
>
> synchronized在修饰方法和代码块在字节码上实现方式有很大差异，但是内部实现还是基于对象头的MarkWord来实现的。 JDK1.6之前synchronized使用的是重量级锁，**JDK1.6之后进行了优化，拥有了无锁->偏向锁->轻量级锁->重量级锁的升级过程**，而不是无论什么情况都使用重量级锁。
>
> - **偏向锁**：适用于单线程适用的情况，在不存在锁竞争的时候进入同步方法/代码块则使用偏向锁。
> - **轻量级锁**：适用于竞争较不激烈的情况(这和乐观锁的使用范围类似)， 存在竞争时升级为轻量级锁，轻量级锁采用的是自旋锁，如果同步方法/代码块执行时间很短的话，采用轻量级锁虽然会占用cpu资源但是相对比使用重量级锁还是更高效。
> - **重量级锁**：适用于竞争激烈的情况，如果同步方法/代码块执行时间很长，那么使用轻量级锁自旋带来的性能消耗就比使用重量级锁更严重，这时候就需要升级为重量级锁。

- 偏向锁：Mark Word_存储的是偏向的**线程ID**。有且只有**一个线程**来访问。
- 轻量锁：Mark Word_存储的是指向线程中_Lock Record_的指针。**有多个线程A、B来交替访问**
- 重量：Mark Word_存储的是指向堆中_Monitor_对象的指针。**竞争激烈，多个线程来访问

![](http://image.clickear.top/20220926112300.png)

![](http://image.clickear.top/20220926112939.png)

> [!NOTE] 升级过程。
>  1. 首次访问，直接使用偏向锁。将线程id保存在对象头中，下次要锁时，直接进行判断。
>  2. 如果有其它线程访问，尝试CAS来替换线程id，如果原来线程存在，则升级为轻量级锁。

### 偏向锁(仅一个线程，对象头存线程id)

- 当一段同步代码一直被同一个线程多次访问，由于只有一个线程那么该线程在后续访问时便会自动获得锁
- 同一个老顾客来访，直接老规矩行方便.

只需要在锁第一次被拥有的时候，记录下偏向线程ID。这样偏向线程就一直持有着锁(后续这个线程进入和退出这段加了同步锁的代码块时，**不需要再次加锁和释放锁。**而是直接比较对象头里面是否存储了指向当前线程的ID(_偏向锁_))。

- **如果相等**表示偏向锁是偏向于当前线程的，就不需要再尝试获得锁了，直到竞争发生才释放锁。以后每次同步，检查锁的偏向线程ID与当前线程ID是否一致，如果一致直接进入同步。无需每次加锁解锁都去CAS更新对象头。_**如果自始至终使用锁的线程只有一个**_，很明显偏向锁几乎没有额外开销，性能极高。
- **如果不相等**意味着发生了竞争，锁已经不是总是偏向于同一个线程了，这个时候会尝试使用CAS来替换MarkWord里面的线程ID为新的线程ID
    - **竞争成功**：表示之前的线程不存在了，MarkWord里面的线程ID为新的线程ID，锁不会升级，仍为偏向锁
    - **竞争失败**：这时候可能需要升级变为轻量级锁，才能保证线程间公平竞争锁。

### 轻量级锁（多线程交替进入临界区，自旋，对象头存Lock Record指针，）

有线程来参与锁的竞争，但是获取锁的冲突时间极短，**本质就是自旋锁**  
在没有多线程竞争的前提下，**通过CAS减少**重量级锁使用操作系统互斥量产生的性能消耗，**说白了先自旋再阻塞**。

JVM会为每个线程在当前线程的栈帧中创建用于存储锁记录的空间，官方成为`Displaced Mark Word`。若一个线程获得锁时发现是轻量级锁，会把锁的MarkWord复制到自己的Displaced Mark Word里面。然后线程尝试用CAS将锁的MarkWord替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示Mark Word已经被替换成了其他线程的锁记录，说明在与其它线程竞争锁，当前线程就尝试使用自旋来获取锁。

- 争夺轻量级锁失败时，自旋尝试抢占锁。

### 重量级锁(多线程，存的是Monitor的指针)

Java中synchronizedi的重量级锁，是基于进入和退出Monitor>对象实现的。在编译时会将同步块的开始位置插入monitor enter指令，在结束位置插入monitor exit指令。  
当线程执行到monitor enter指令时，会尝试获取对象所对应的Monitor所有权，如果获取到了，即获取到了锁，会在Monitor的owner中存放当前线程的id,这样它将处于锁定状态，除非退出同步块，否则其他线程无法获取到这个Monitor。

### 对比

|    锁    |                             优点                             |                      缺点                      |              适用场景              |
| :------: | :----------------------------------------------------------: | :--------------------------------------------: | :--------------------------------: |
|  偏向锁  | 加锁和解锁不需要额外的消耗，和执行非同步方法相比仅存在纳秒级的差距 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗 |   适用于有一个线程访问同步块场景   |
| 轻量级锁 |           竞争的线程不会阻塞，提高了程序的响应速度           | 如果始终得不到锁竞争的线程，使用自旋会消耗CPU  | 追求响应时间，同步块执行速度非常快 |
| 重量级锁 |               线程竞争不使用自旋，不会消耗CPU                |             线程阻塞，响应时间缓慢             |   追求吞吐量、同步块执行速度较长   |

## hashcode哪里去了？

> [!NOTE] 笔记  
> 首先，hashcode是在调用hashcode()方法时，才会生成，即惰性生成。而不是在new时，就创建好。

- **在无锁状态下**，Mark Word中可以存储对象的identity hash code值。当对象的hashCode()方法第一次被调用时，JVM会生成对应的identity hash code值并将该值存储到Mark Word中。
- **对于偏向锁**，在线程获取偏向锁时，会用Thread ID和epoch值覆盖identity hash code所在的位置。如果一个对象的hashCode()方法己经被调用过一次之后，这个对象不能被设置偏向锁。因为如果可以的化，那Mark Word中的identity hash code必然会被偏向线程ld给覆盖，这就会造成同一个对象前后两次调用hashCode()方法得到的结果不一致。
 - **升级为轻量级锁时**，JVM会在当前线程的栈帧中创建一个锁记录(Lock Record)空间，用于存储锁对象的Mark Word拷贝，该拷贝中可以包含identity hash code,所以**轻量级锁可以和identity hash code共存**，哈希码和GC年龄自然保存在此，释放锁后会将这些信息写回到对象头。
- **升级为重量级锁后**，Mark Word保存的重量级锁指针代表重量级锁的ObjectMonitor类里有字段记录非加锁状态下的Mark Word,锁释放后也会将信息写回到对象头。

偏向锁之前，已经调用过hashcode方法，只能升级到 轻量级锁。  
在已偏向的线程中，调用hashcode方法，会升级成 重量级锁。不知道为什么不是轻量级锁，可能已偏向升级到轻量级成本很高？

## wait为什么需要释放锁？

当线程A调用[[synchronized]]方法，锁对象。如果调用wait时，不释放锁，进入阻塞状态的话，其它线程就无法进入到[[synchronized]]代码块了，无法调用notify方法，此时线程A无法被唤醒。  
wait的实现伪代码

```
wait(){
// 释放锁
// 阻塞，等待被其它线程notify
// 重新拿锁。
}
```

## wait和notify存在问题

```java
public void enqueue(){
	synchronized(queue){
		while(queue.full()) queue.wait();
		// .... 入队列
		queue.notify();// 通知消费者，队列中有数据
	}
}

public void dequeue(){
	synchronized(queue){
			while(queue.empty()) queue.wait();
			// 。。。。 出队列
			queue.notify();// 通知生产者，队列有空位，可以放了。
	}

}
```

这里有个明显问题，就是wait和notify使用的是同一个对象，无法区分队列满和空2个条件。会导致生产者原本只想通知消费者，但它把其它生产者也通知了。  
这正是condition要解决的。

```java

public Lock lock = new ReentrantLock();
public Condition notEmpty = lock.newCondition();
public Condition notFull = lock.newCondition();

public void enqueue(){
	lock.lockInterruptibly();
	while(queue.full()){
		 notFull.await();// put的时候队列满了。
	}
		// .... 入队列
		notEmpty.signal();// 通知消费者，队列中有数据
	}
}

public void dequeue(){
	synchronized(queue){
			while(queue.empty()) notEmptry.wait();
			// 。。。。 出队列
			notFull.signal();// 通知生产者，队列有空位，可以放了。
	}

}
```

# 资料

[Synchronized底层实现，锁升级原理 | Java识堂](https://www.javashitang.com/md/concurrent/Synchronized%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0%EF%BC%8C%E9%94%81%E5%8D%87%E7%BA%A7%E5%8E%9F%E7%90%86.html)  
[Synchronized与锁升级 - YuBlog](https://xiaoyu72.com/articles/b04bf8ad/)
