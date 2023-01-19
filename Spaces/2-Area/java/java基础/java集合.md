---
title: java集合
date created: 2023-01-19
date modified: 2023-01-19
---

[【集合系列】- 初探 java 集合框架图 | Just Do Java](https://www.justdojava.com/2019/09/16/java-collection-1/)

Collections

![](http://image.clickear.top/20230119164335.png)

Map  
![](http://image.clickear.top/20230119164343.png)

+ Collections
	+ List 有序可重复集合
		+ ArrayList 数组
		+ LinkedList 链表
		+ Vector 用的少了，线程安全，动态数组。性能有问题
		+ Stack，栈(先进后出)，是Vector 也用得少，可以用ArrayDeque双端队列替代
	+ Queue，队列集合
		+ ArrayDeque双端队列，具有队列和栈的特性
		+ LinkedList，有是Deque的实现类。双向链表
		+ PriorityQueue，排序队列
	+ Set， 无序，不可重复集合
		+ HashSet, 底层是通过[[HashMap]]来实现。
		+ LinkedHashSet， 底层是通过LinkedHashMap实现
		+ TreeSet，底层是通过TreeMap实现。
+ Map
	+ [[HashMap]]
	+ [[LinkedHashMap]]
	+ [[TreeMap]]
	+ HashTable 线程安全。所有方法加 synchronized
	+ Properties，继承自HashTable
