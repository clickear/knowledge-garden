---
date created: 2022-09-08
date modified: 2022-09-08
title: redolog和undolog的区别
---

二种日志均可以视为一种恢复操作，

+ [[redolog]]g是**恢复提交事务修改的页操作**，而[[undolog]]是**回滚行记录到特定版本**。，
+ 记录内容: [[redolog]]是物理日志，记录页的物理修改操作，而[[undolog]]是逻辑日志，根据每行记录进行记录
