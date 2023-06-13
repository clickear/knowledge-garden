---
title: Log-structured
date created: 2023-06-12
date modified: 2023-06-13
---

> [!TIP] 技巧💡  
> Log-structured]]存日志，使用[[out-place-update-structure]]异地更新结构，会将更新的内容存到到新的位置，而不是覆盖。如 [[LSM树]]

log-structured 只存储日志记录。不是存具体的数据，如果需要数据，只能通过日志来计算出来。

1. 增删改，都需要新增一条日志记录。不用删除具体的数据，会变得很快。因为是[[顺序写]]的原因。
2. 有得有失，因为每次要查询的时候，都得重新计算值。所以查询慢。  
![image.png](http://image.clickear.top/20230612225259.png)

优化方向:

1. 压缩日志，比如有2个日志，一个是update a = 5, 后面是 update a = 6; 则可以直接删除 a=5那条记录。
2. 分级压缩, rosedb 和leveldb 的主要方法
3. ![image.png](http://image.clickear.top/20230612225758.png)

## [[LSM树]]
