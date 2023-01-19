---
title: final、fianally、finalize的区别
date created: 2023-01-19
date modified: 2023-01-19
---

> [!TIP] 技巧💡  
>  final: 一定程度上的**不可变**，有点只读的意思。可修饰类(不可继承)、变量(不可重新赋值)、方法(不可重载)  
finnally: try-finally 。用于资源释放等。但不建议使用  
finalize: Object 的方法，不建议重载。性能触发具有不确定性。一般使用 [[虚引用]]代替进行资源释放

## final

编程的其中一个原则，就是尽量内聚。合适的控制类的范围。  
finnal 和 不可变[[immutable]]是不一样的。finnal可以认为只是只读，不能对变量进行重新赋值。

```java
final List<String> strList = new ArrayList<>();
// 可以执行成功，并非不可变。只是strList不能重新赋值

strList.add("Hello");

strList.add("world");

// jdk9的方法，返回的是 immutable
List<String> unmodifiableStrList = List.of("hello", "world");
// 会报错，不能进行add操作
unmodifiableStrList.add("again");
```

## finnally

一般用于异常之后，资源释放，锁释放等

```java
try(){

} finally{
	// 锁释放
}
```

释放资源，也可使用 try-with-resource  
堆外内存，可考虑使用 [[虚引用]]

## finalize

Object的一个方法，不建议使用。是在对象被回收时，会调用的方法。会导致jvm回收时间变长等情况。具有不确定性。
