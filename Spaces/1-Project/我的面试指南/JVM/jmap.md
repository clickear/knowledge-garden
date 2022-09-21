---
date created: 2022-09-06
date modified: 2022-09-06
title: jmap
---

> [!INFO] jmap, 堆操作  
>  jmap执行时，会STW，不能在生产环境使用。

``` shell
java -Xms20M -Xmx20M -XX:+UseParallelGC -XX:+HeapDumpOnOutOfMemoryError com.mashibing.jvm.gc.T15_FullGC_Problem01;

jmap - histo 4655 | head -20，查找有多少对象产生
```

线上系统，内存特别大，jmap执行期间会对进程产生很大影响，甚至卡顿（电商不适合）  
1：设定了参数HeapDump，OOM的时候会自动产生堆转储文件（不是很专业，因为多有监控，内存增长就会报警）  
2：<font color='red'>很多服务器备份（高可用），停掉这台服务器对其他服务器不影响</font>  
3：在线定位(一般小点儿公司用不到)  
4：在测试环境中压测（产生类似内存增长问题，在堆还不是很大的时候进行转储）

使用MAT / jhat /jvisualvm 进行dump文件分析[https://www.cnblogs.com/baihuitestsoftware/articles/6406271.html](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fbaihuitestsoftware%2Farticles%2F6406271.html) jhat -J-mx512M xxx.dump http://192.168.17.11:7000 拉到最后：找到对应链接 可以使用OQL查找特定问题对象
