---
title: HashMap
date created: 2023-01-19
date modified: 2023-01-20
---

[toc]

## HashMap的底层实现

线程不安全，如需要线程安全，使用[[ConcurrentMap]]

|      | JDK1.7              | JDK1.8                                                              |
|:---- |:------------------- |:------------------------------------------------------------------- |
| 结构 | 数组 + 链表(拉链法) | 数组 + 链表 + 红黑树(链表长度>8)                                    |
| 扩容 | 根据hash频繁迁移    | 只需看mask多一位，即只需要看看原来的hash值新增的那个bit是1还是0就好 |
| 线程安全     |         头插法，死循环问题            |         尾插法，存在覆盖问题                                                            |

## 底层结构

+ 1.7：  
![](http://image.clickear.top/20230120112851.png)

+ 1.8:  
![](http://image.clickear.top/20230120112923.png)

+ 1. 根据hash算法，找到数组的index,
+ 2. hash冲突时，--> 链表
+ 3. 链表长度 > 8
	+ 数组长度 < 64。选择先进行数组扩容
	+ 数组长度 >=64, 转换成红黑树

### 扩容

+ capacity 即容量, 默认16.
+ loadFactor 加载因子, 默认是0.75
+ threshold 阈值. 阈值=容量*加载因子. 默认12. 当元素数量超过阈值时便会触发扩容.

> [!TIP] 技巧💡  
> 扩容时机， 当元素数量超过阈值时便会触发扩容. 每次扩容的容量都是之前容量的**2倍 **

JDK1.7:  
所有元素需要遍历过去，重新计算位置。进行迁移。

JDK1.8:

> [!TIP] 技巧💡  
> 以2的幂次方扩容，即只需要判断新增位的bit即可。要么不动，要么转移到新创建的分区。

由于数组的容量是以2的幂次方扩容的, 那么一个Entity在扩容时, 新的位置要么在原位置, 要么在原长度+原位置的位置。  
![](http://image.clickear.top/20230120114228.png)

![](http://image.clickear.top/20230120114236.png)

### 线程安全

jdk1.7:  
头插法，死循环问题

jdk1.8:  
尾插法，不存在死循环，但是存在数据覆盖问题

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null) // 如果没有hash碰撞则直接插入元素
            tab[i] = newNode(hash, key, value, null);
        else {
            // ....
        }
        ++modCount;
        if (++size > threshold)  //非原子操作，会有问题
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

存在原因:

1. 首次插入时，因为判断非线程安全，可能A线程判断为空数据，准备插入节点，B线程也判断的情况。此次2个线程都会创建节点。
2. ++size非原子性，会带来问题

## 参考资料

[Java集合常见面试题总结(下) | JavaGuide(Java面试+学习指南)](https://javaguide.cn/java/collection/java-collection-questions-02.html#hashmap-%E7%9A%84%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0)  
[Java 8系列之重新认识HashMap - 美团技术团队](https://tech.meituan.com/2016/06/24/java-hashmap.html)
