---
date created: 2022-09-02
date modified: 2022-09-02
title: JAVA中各变量分配空间和初始化时机
---

java中存在各种变量，我们来讨论下各变量的分配空间时机和初始化时机。

``` java
public TestClass(){  
	private final Long finalLong = 5;  
	// 这里被final修饰的变量finalStr，直接成为常量，编译时就会被分配为5;  
	private static Long staticLong = 1;  
	// 静态变量staticLongr， 静态字段在[[准备]]阶段只会被赋值为0，[[初始化]]时才会被赋值为1。  
	private Long memberLong = 1;  
	// 成员变量，在new的时候分配空间和初始化。  
	private static void newInstance(){  
		TestClass testClass = new TestClass();  
		// testClass 是分配在栈中，指向分配在TestClass的堆对象。  
	}  
}
```

final修饰的变量 编译时，变成常量  
类静态变量 在[[准备]]阶段赋值为0， 在[[初始化]]阶段赋值为1 。 分配在方法区  
实例变量 在new的时候，分配空间和初始化，在JAVA堆  
局部变量 在创建的时候赋值。 分配在栈中
