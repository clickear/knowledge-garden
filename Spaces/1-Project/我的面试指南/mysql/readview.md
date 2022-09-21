---
date created: 2022-09-09
date modified: 2022-09-09
title: readview
---

> 什么是Read View，说白了Read View就是事务进行快照读操作的时候生产的读**视图**(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID(当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大)。

> [!TIP] 技巧💡  
> 在RC [[读已提交]]隔离级别下，是每个快照读都会生成并获取最新的Read View  
> 在RR [[可重复读]]隔离级别下，则是同一个事务中的第一个快照读才会创建Read View, 之后的快照读获取的都是同一个Read View

Read View主要是用来做可见性判断的, 即当我们某个事务执行快照读的时候，对该记录创建一个Read View读视图，把它比作条件用来判断当前事务能够看到哪个版本的数据，既可能是当前最新的数据，也有可能是该行记录的undo log里面的某个版本的数据。

 在[[快照读]]时，把未提交事务列表记录下来。  
	- 在[[可重复读]]隔离级别下，使用的是同一个readview。  
	- 在[[读已提交]]隔离级别下，会在每次进行快照读时，进行记录。  
readview包含3部分:

- **trx_list** 未提交事务ID列表，用来维护Read View生成时刻系统正活跃的事务ID。  
- **up_limit_id** 记录trx_list列表中事务ID最小的ID。
- **low_limit_id** ReadView生成时刻系统尚未分配的下一个事务ID，也就是目前已出现过的事务ID的最大值+1。

## 可见性分析

结合[[MVCC]]可知，会有个[[undolog]]链，首节点是最新的，尾节点是旧记录。在[[快照读]]时，会获取[[readview]]。readview是使用本事务第一次获取的readview还是获取最新的readvew，需要根据[[事务隔离级别]]，[[可重复读|RR]]使用第一次获取的readview，[[读已提交|RC]]则是每次都进行获取最新readview

步骤:  
DB_TRX_ID， 当前undolog记录的事务id。

- 首先比较DB_TRX_ID < up_limit_id, 如果小于，则当前事务能看到DB_TRX_ID 所在的记录，如果大于等于进入下一个判断
- 接下来判断 DB_TRX_ID 大于等于 low_limit_id , 如果大于等于则代表DB_TRX_ID 所在的记录在Read View生成后才出现的，那对当前事务肯定不可见，如果小于则进入下一个判断
- 判断DB_TRX_ID 是否在活跃事务之中，trx_list.contains(DB_TRX_ID)，  
  如果在，则代表我Read View生成时刻，你这个事务还在活跃，还没有Commit，你修改的数据，我当前事务也是看不见的；  
  如果不在，则说明，你这个事务在Read View生成之前就已经Commit了，你修改的结果，我当前事务是能看见的

在不可见的情况下，都会根据根据回滚指针，进行下一个undolog记录的判断。回到第一步，直至成功获取到可见的数据。  
![](http://image.clickear.top/20220909113423.png)
