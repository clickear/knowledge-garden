---
title: exception和error的区别
date created: 2023-01-19
date modified: 2023-01-19
---

> [!TIP] 技巧💡  
>  Error 是系统、虚拟机出错了。我们不用处理也处理不了  
>  Exception 是程序的异常，需要我们处理。其中RuntimeException 是我们经常使用的

NoClassDefFoundError（编译通过了，但是jvm加载时，发现少类。比如 A.class 依赖B.class，编译后将B.class删除，又比如，B.class加载时，初始化失败，在public static int i = 0/23; ） 和 ClassNotFoundException （运行时少类，比如使用了 Class.forName 去加载一个未在classpath的类）

![](http://image.clickear.top/20230119105344.png)  

## try 和try-with-resource的使用

```java
try (BufferedReader br = new BufferedReader(…);
BufferedWriter writer = new BufferedWriter(…)) {// Try-with-resources
// do something
catch ( IOException | XEception e) {// Multiple catch
// Handle it
}finally{

}
```

try-with-resource 在编译时期，会自动生成相应的处理逻辑，比如，自动按照约定俗成 close 那些扩展了 AutoCloseable 或者 Closeable 的对象。

## 注意点

1. 异常不能直接吃掉，要打印日志。不能使用 e.printStackTrace(); 而应该使用sl4j 或者其它日志组件。 并且最好将上下文数据打印，方便排查
