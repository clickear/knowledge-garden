---
date created: 2022-09-13
date modified: 2023-02-15
title: mysql调优
---

## 优化步骤

步骤：  
1.开启慢查询日志,设置阈值,比如超过5秒钟的就是慢SQL,并将它抓取出来;  
2.EXPLAIN+慢SQL分析;  
3.SHOW profile,查询SQL在MySQL服务器里面的执行细节和生命周期情况  
4.具体优化

## explain[^1]

语法explain select * from xxl_job_log l where l.job_id in (select id from xxl_job_info)  

![](http://image.clickear.top/20220913175031.png)  

## type

> [!TIP] 技巧💡  
> system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL ）。最少要到range级别

1) system：表只有一行记录（等于系统表），是 const 类型的特例，平时不会出现。[^2]

2) const：**唯一性索引的等值查找**,表示通过索引一次就找到了，const 用于比较 primary key 或 unique 索引，因为只要匹配一行数据，所以很快，如将主键置于 where 列表中，mysql 就能将该查询转换为一个常量。可以认为是通过唯一性索引[^3]进行查找。

3) eq_ref：**唯一性索引的等值关联**，唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配，常见于主键或唯一索引扫描。[^eq_ref和const的区别]

4) ref：**非唯一性索引的等值扫描**,非唯一性索引扫描，范围匹配某个单独值得所有行。本质上也是一种索引访问，他返回所有匹配某个单独值的行，然而，它可能也会找到多个符合条件的行，多以他应该属于查找和扫描的混合体

5) range：**范围查找，不指定唯一索引还是普通索引**只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现了between、<、>、in等的查询，这种范围扫描索引比全表扫描要好，因为它只需开始于索引的某一点，而结束于另一点，不用扫描全部索引

6) index：Full Index Scan，index于ALL区别为index类型只遍历索引树。通常比ALL快，因为索引文件通常比数据文件小。（也就是说虽然all和index都是读全表，但index是从索引中读取的，而all是从硬盘中读的）

7) ALL：Full Table Scan，将遍历全表找到匹配的行

缺点：  

- EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况  
- EXPLAIN不考虑各种Cache
- EXPLAIN不能显示MySQL在执行查询时所作的优化工作  
- 部分统计信息是估算的，并非精确值  
- EXPLAIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划

## extra(扩展信息)

```sql
CREATE TABLE `user` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `user_code` varchar(20) NOT NULL DEFAULT '' COMMENT '唯一索引',
  `name` varchar(20) NOT NULL DEFAULT '' COMMENT '名称',
  `age` bigint(20) NOT NULL DEFAULT '0' COMMENT '年龄',
  `birth_date` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '生日',
  `sex` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '性别',
  PRIMARY KEY (`id`),
  UNIQUE KEY uniq_user_code(`user_code`),
  KEY `idx_age_name` (`name`,`age`),
  KEY `idx_birth_date` (`birth_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='用户';
```

### using where (有点类型，是否服务端过滤)

 > 通常来说，意味着全表扫描或者在查找使用索引的情况下，但是还有查询条件不在索引字段当中。

``` sql
select * from user where sex = 1; -- 不走索引
```

### using index

[[索引覆盖]]，不进行[[回表]]。可以索引的字段信息，就以及满足查询需求了，不需要进行查其他字段信息。索引包含（或者说覆盖）所有需要查询的字段。

  
  ```sql
select * from user where age = 1; -- extra 为null  
select age from user where age = 1; -- extra 为using index。 因为直接使用索引就可以，不用进行查询
select id,user_code from user where user_code = 't'; -- 与是否为使用唯一索引无关
```
  

### using index; using where

索引覆盖和where 的结合。

```sql
-- 索引覆盖
select name from user where name = 'xx' and age = 12;

-- using index, using where 这个为什么不是索引下推，很迷惑。我理解是这里不需要回表，可以进行索引覆盖，优先级比索引下推高。
select name from user where name = 'xx' and age > 12;

-- 索引下推(复合索引情况下使用)， using index condition
select * from user where name = 'xx' and age > 12;
```

### using index condition

[[索引下推]]  
如name = 'xx' and age > 12 为示例，(如果是name = 'xx' and age = 12，此时直接使用索引了，也就没有下推的说法了。)  
无索引下推: 先定位name的索引，然后回表查询完整记录，在到server层过滤数据。  
有索引下推: 先定位name的索引，在索引中，记录判断 age > 12的数据，然后回表查询到完整记录。在server层不用进行过滤了。

```sql
-- 索引
select * from user where name = 'xx' and age = 12;

-- 索引覆盖
select name from user where name = 'xx' and age = 12;

-- 索引下推(复合索引情况下使用)， using index condition
select * from user where name = 'xx' and age > 12;

-- using index, using where 这个为什么不是索引下推，很迷惑。我理解是这里不需要回表，可以进行索引覆盖，优先级比索引下推高。
select name from user where name = 'xx' and age > 12;

-- 索引下推（索引覆盖不了sex字段） + where ， using index condition, using where 
select name from user where name = 'xx' and age > 12 and sex = 1;

-- 索引下推（索引覆盖不了sex字段） + where ， using index condition, using where 
select * from user where name = 'xx' and age > 12 and sex = 1;

```

## 总结

> [!ABSTRACT] 结论  
>  在范围查询时，索引覆盖的优先级 > 索引下推。 即能索引覆盖在server层过滤，就尽量使用索引覆盖

> [!TIP] 技巧💡
>
> 等值查询时(name = 'xxx' and age = 12)  
> --索引能够覆盖查询，(select name) **覆盖索引, using index**  
> --索引不能覆盖查询  
>----索引不需要服务端过滤（select \* ) *NULL*  
>----索引需要服务端过滤(and sex = 1) *using where *  
>范围查询(name = 'xxx' and age > 12)  
>--索引能够覆盖查询，(select name) **using index; using where**  
>--索引不能覆盖查询  
>----索引不需要服务端过滤(select \* ) ** 索引下推, using index condition ;**  
>---- 索引需要服务端过滤(and sex = 1) **索引下推 + where, using index condition; using where**

		

   

## 资料

[MySQL 三万字精华总结 + 面试100 问，和面试官扯皮绰绰有余（收藏系列） - 掘金](https://juejin.cn/post/6850037271233331208)

[SQL中的where条件，在数据库中提取与应用浅析](https://www.jianshu.com/p/89ec04641e72)

[^1]: [MySQL 执行计划中Extra(Using where,Using index,Using index condition,Using index,Using where)的浅析 - 潇湘隐者 - 博客园](https://www.cnblogs.com/kerrycode/p/9909093.html)

[^2]: 相当于表中只有一条数据，由于innodb无法准确记录数据库中的表数量，只是预估，所以即使是只有一条数，显示的type也只是const。[Why is const rather than system of type in mysql explain? - Stack Overflow](https://stackoverflow.com/questions/67527921/why-is-const-rather-than-system-of-type-in-mysql-explain)

[^3]: 唯一性索引，指主键或者唯一索引

[^eq_ref和const的区别]: eq_ref和const，虽然都是唯一性索引扫描和查找，但是const是在表中使用，而eq_ref主要是用于关联表的时候。即区分关联表和普通表的情况
