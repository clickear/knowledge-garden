---
title: Tuple-oriented
date created: 2023-06-13
date modified: 2023-06-13
---

> [!TIP] 技巧💡  
> [[Tuple-oriented]]存数据，使用[[In-place update structure]]就地更新结构，如[[B+树]]。是直接覆盖旧记录来存储更新内容。

## 存储布局

### strawman idea(基本不用)

在 header 中记录 tuple 的个数，然后不断的往下 append 即可。

![image.png](http://image.clickear.top/20230612224329.png)  
缺点:

1. 容易碎片化，比如要删除 tuple#2的记录，就容易出现空余位置(碎片)，每次遍历查询空余位置，浪费时间
2. 变长的tuple内容长度，没办法解决。只能取最大长度？浪费空间

### slotted pages——有点像操作系统里的页表（采用）

header 中的 slot array 记录每个 slot 的信息，如大小、位移等。下面tuple的长度不必限制都是等长的，完全可以变长。在 slot array 中新增一条记录，记录着改记录的入口地址，slot array 与 data 从 page 的两端向中间生长，二者相遇时，就认为这个 page 已经满了。  
![image.png](http://image.clickear.top/20230612224726.png)

如何定义一条记录呢？page_id + offset/slot 来表示 。record Id，主要是数据库内部使用的，外部应用程序应该使用 主键来标识数据。而不是依赖具体的record Id  
![image.png](http://image.clickear.top/20230612224845.png)

## [[tuple-layout]]
