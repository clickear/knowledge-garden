---
date created: 2022-09-13
date modified: 2022-09-13
title: 资料
---

## 优化步骤

步骤：  
1.开启慢查询日志,设置阈值,比如超过5秒钟的就是慢SQL,并将它抓取出来;  
2.EXPLAIN+慢SQL分析;  
3.SHOW profile,查询SQL在MySQL服务器里面的执行细节和生命周期情况  
4.具体优化

## explain

语法explain select * from xxl_job_log l where l.job_id in (select id from xxl_job_info)  

![](http://image.clickear.top/20220913175031.png)  

> [!TIP] 技巧💡  
> system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL ）。最少要到range级别

1) system：表只有一行记录（等于系统表），是 const 类型的特例，平时不会出现

2) const：表示通过索引一次就找到了，const 用于比较 primary key 或 unique 索引，因为只要匹配一行数据，所以很快，如将主键置于 where 列表中，mysql 就能将该查询转换为一个常量

3) eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描

4) ref：非唯一性索引扫描，范围匹配某个单独值得所有行。本质上也是一种索引访问，他返回所有匹配某个单独值的行，然而，它可能也会找到多个符合条件的行，多以他应该属于查找和扫描的混合体

5) range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询，这种范围扫描索引比全表扫描要好，因为它只需开始于索引的某一点，而结束于另一点，不用扫描全部索引

6) index：Full Index Scan，index于ALL区别为index类型只遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。（也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的）

7) ALL：Full Table Scan，将遍历全表找到匹配的行

缺点：  

- EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况  
- EXPLAIN不考虑各种Cache  
- EXPLAIN不能显示MySQL在执行查询时所作的优化工作  
- 部分统计信息是估算的，并非精确值  
- EXPLAIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划

## 优化方式

# 资料

[MySQL 三万字精华总结 + 面试100 问，和面试官扯皮绰绰有余（收藏系列） - 掘金](https://juejin.cn/post/6850037271233331208)
