---
aliases:
  - 二进制日志
date created: 2022-09-08
date modified: 2022-09-09
title: binlog
---

> [!TIP] 记录整个操作过程💡
>  

MySQL的二进制日志（binary log）是一个二进制文件，主要记录所有数据库表结构变更（例如CREATE、ALTER TABLE…）以及表数据修改（INSERT、UPDATE、DELETE…）的所有操作。二进制日志（binary log）中记录了对MySQL数据库执行更改的所有操作，并且记录了语句发生时间、执行时长、操作数据等其它额外信息，但是它不记录SELECT、SHOW等那些不修改数据的SQL语句

## 作用

恢复（recovery）：某些数据的恢复需要二进制日志。例如，在一个数据库全备文件恢复后，用户可以通过二进制日志进行point-in-time的恢复

复制（replication）：其原理与恢复类似，通过复制和执行二进制日志使一台远程的MySQL数据库（一般称为slave或者standby）与一台MySQL数据库（一般称为master或者primary）进行实时同步。

审计（audit）：用户可以通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入攻击

## binlog

binlog格式分为: STATEMENT、ROW和MIXED三种，详情如下:

- STATEMENT

STATEMENT格式的binlog记录的是数据库上执行的原生SQL语句。这种方式有好处也有坏处。

好处就是相当简单，简单地记录和执行这些语句，能够让主备保持同步，在主服务器上执行的SQL语句，在从服务器上执行同样的语句。另一个好处是二进制日志里的时间更加紧凑，所以相对而言，基于语句的复制模式不会使用太多带宽，同时也节约磁盘空间。并且通过mysqlbinlog工具容易读懂其中的内容。

坏处就是同一条SQL在主库和从库上执行的时间可能稍微或很大不相同，因此在传输的二进制日志中，除了查询语句，还包括了一些元数据信息，如当前的时间戳。即便如此，还存在着一些无法被正确复制的SQL。例如，使用INSERT INTO TB1 VALUE(CUURENT_DATE())这一条使用函数的语句插入的数据复制到当前从服务器上来就会发生变化。存储过程和触发器在使用基于语句的复制模式时也可能存在问题。另外一个问题就是基于语句的复制必须是串行化的。这要求大量特殊的代码，配置，例如InnoDB的next-key锁等。并不是所有的存储引擎都支持基于语句的复制。

- ROW

从MySQL5.1开始支持基于行的复制，也就是基于数据的复制，基于行的更改。这种方式会将实际数据记录在二进制日志中，它有其自身的一些优点和缺点，最大的好处是可以正确地复制每一行数据。一些语句可以被更加有效地复制，另外就是几乎没有基于行的复制模式无法处理的场景，对于所有的SQL构造、触发器、存储过程等都能正确执行。主要的缺点就是二进制日志可能会很大，而且不直观，所以，你不能使用mysqlbinlog来查看二进制日志。也无法通过看二进制日志判断当前执行到那一条SQL语句了。

现在对于ROW格式的二进制日志基本是标配了，主要是因为它的优势远远大于缺点。并且由于ROW格式记录行数据，所以可以基于这种模式做一些DBA工具，比如数据恢复，不同数据库之间数据同步等。

- MIXED

MIXED也是MySQL默认使用的二进制日志记录方式，但MIXED格式默认采用基于语句的复制，一旦发现基于语句的无法精确的复制时，就会采用基于行的复制。比如用到UUID()、USER()、CURRENT_USER()、ROW_COUNT()等无法确定的函数。

## 配置

- max_binlog_size

可以通过max_binlog_size参数来限定单个binlog文件的大小（默认1G），如果当前binlog文件的大小达到了参数指定的阈值，会创建一个新的binlog文件作为当前活跃的binlog文件，后续所有对数据库的修改都会记录到新的binlog文件中。

对于binlog文件的大小，有个需要注意的地方是，binlog文件可能会大于max_binlog_size参数设定的阈值。由于一个事务所产生的所有事件必须记录在同一个binlog文件中，所以即使binlog文件的大小达到max_binlog_size参数指定的大小，也要等到当前事务的所有事件全部写入到binlog文件中才能切换，这样就会出现binlog文件的大小大于max_binlog_size参数指定的大小的情况。

- binlog_cache_size

当使用事务的表存储引擎（如InnoDB存储引擎）时，所有未提交（uncommitted）的二进制日志会被记录到一个缓存中去，等该事务提交（committed）时直接将缓冲中的二进制日志写入二进制日志文件，而该缓冲的大小由binlog_cache_size决定，默认大小为32K。此外，binlog_cache_size是基于会话（session）的，也就是说，当一个线程开始一个事务时，MySQL会自动分配一个大小为binlog_cache_size的缓存，因此该值的设置需要相当小心，不能设置过大。当一个事务的记录大于设定的binlog_cache_size时，MySQL会把缓冲中的日志写入一个临时文件中，因此该值又不能设得太小。通过SHOW GLOBAL STATUS命令查看binlog_cache_use、binlog_cache_disk_use的状态，可以判断当前binlog_cache_size的设置是否合适。binlog_cache_use记录了使用缓冲写二进制日志的次数，binlog_cache_disk_use记录了使用临时文件写二进制日志的次数。

- sync_binlog

在MySQL 5.7之前版本默认情况下，二进制日志并不是在每次写的时候同步的磁盘（用户可以理解为缓冲写）。因此，当数据库所在的操作系统发生宕机时，可能会有最后一部分数据没有写入二进制文件中，这会给恢复和复制带来问题。参数sync_binlog=[N]中的N表示每提交多少个事务就进行binlog刷新到磁盘。如果将N设为1，即sync_binlog=1表示采用同步写磁盘的方式来写二进制日志，每次事务提交时就会刷新binlog到磁盘；sync_binlog为0表示刷新binlog时间点由操作系统自身来决定，操作系统自身会每隔一段时间就会刷新缓存数据到磁盘；sync_binlog为N表示每N个事务提交会进行一次binlog刷新。如果使用Innodb存储引擎进行复制，并且想得到最大的高可用性，需要将此值设置为1。不过该值为1时，确时会对数据库IO系统带来一定的开销。

但是，即使将sync_binlog设为1，还是会有一种情况导致问题的发生。当使用InnoDB存储引擎时，在一个事务发出COMMIT动作之前，由于sync_binlog为1，因此会将二进制日志立即写入磁盘。如果这时已经写入了二进制日志，但是提交还没有发生，并且此时发生了宕机，那么在MySQL数据库下次启动时，由于COMMIT操作并没有发生，这个事务会被回滚掉。但是二进制日志已经记录了该事务信息，不能被回滚。对于这个问题，MySQL使用了两阶段提交来解决的，简单说就是对于已经写入到binlog文件的事务一定会提交成功， 而没有写入到binlog文件的事务就会进行回滚，从而保证二进制日志和InnoDB存储引擎数据文件的一致性，保证主从复制的安全。

- binlog-do-db&binlog-ignore-db

参数binlog-do-db和binlog-ignore-db表示需要写入或者忽略写入哪些库的二进制日志。默认为空，表示需要同步所有库的日志到二进制日志。

- log-slave-update

如果当前数据库是复制中的slave角色，则它不会将master取得并执行的二进制日志写入自己的二进制日志文件中去。如果需要写入，要设置log-slave-update。如果需要搭建master–>slave–>slave架构的复制，则必须设置该参数。

- binlog-format

binlog_format参数十分重要，用来设置二进制日志的记录格式,详情参考(6.5 binlog格式)

- log_bin_trust_function_creators

默认为OFF，这个参数开启会限制存储过程、Function、触发器的创建
