---
date created: 2022-09-08
date modified: 2023-03-01
title: redolog和undolog的区别
---

二种日志均可以视为一种恢复操作，

+ [[redolog]]g是**恢复提交事务修改的页操作**，而[[undolog]]是**回滚行记录到特定版本**。，
+ 记录内容: [[redolog]]是物理日志，记录页的物理修改操作，而[[undolog]]是逻辑日志，根据每行记录进行记录
+ redolog是循环写的，不持久保存，binlog是追加日志，具有“归档”这个功能，redolog是不具备的
+ redolog只有InnoDB有，别的引擎没有。redolog具有crash safe的功能。而binlog不具有。

为什么会有这2种日志，我举得是因为，mysql设计之初，binlog不具有crash safe的功能。
