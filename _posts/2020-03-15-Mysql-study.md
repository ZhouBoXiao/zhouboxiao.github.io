---
layout:     post
title:      "MySQL相关知识01"
subtitle:   "MySQL的原理、数据结构、引擎、使用、事务隔离级别等"
date:       2020-03-01
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - MySQL
---

## InnoDB引擎

​		InnoDB通过使用多版本并发控制MVCC来获得高并发性，并且实现了SQL的标准的4中隔离级别，默认为可重复读级别。同时，使用一种被称为**next-key locking**的策略来避免幻读现象的产生。除此之外，InnoDB还提供了**insert buffer插入缓冲**、**二次写double write**、**自适应哈希索引adaptive hash index**、**预读 read ahead**等高性能和高可用的功能。

​		对于表中的数据的存储，InnoDB存储引擎采用聚集clustered方式，因此每张表的存储都是按主键的顺序进行存放。如果没有显式地在表定义时指定主键，InnoDB会为每一行生成一个**6字节的ROWID**作为主键。

# 索引

## B+ Tree 

1. 单次请求涉及的磁盘IO次数少（出度d大，且非叶子节点不包含表数据，树的高度小）；
2. 查询效率稳定（任何关键字的查询必须走从根结点到叶子结点，查询路径长度相同），且插入与修改拥有较稳定的对数时间复杂度；（B+树元素自底向上插入）
3. 遍历效率高（从符合条件的某个叶子节点开始遍历即可）；叶子节点之间通过指针来连接，范围扫描将十分简单；
4. B+树的用途[1]
   1. ReiserFS、NSS、XFS、JFS、ReFS和BFS文件系统都使用这种类型的树来进行元数据索引；
   2. BFS还使用B+树来存储目录。NTFS使用B+树来建立目录和与安全相关的元数据索引。EXT4使用extent 树(一个修改过的B+树数据结构)来建立文件的区段索引；
   3. 关系数据库管理系统，如IBM DB2、Informix、Microsoft SQL Server、Oracle 8、Sybase ASE和SQLite[5]支持这种类型的树作为表索引。CouchDB和Tokyo Cabinet等键值数据库管理系统支持这种类型的树来进行数据访问。
5. B+树与B-树
   1. B+树的叶子节点是以链表的形式相互链接；这使得范围查询更简单、更有效。且不会显著增加树的空间消耗或维护成本。B+树的相对于B-树的一个重要优势;
   2. 在 B-树中，**越靠近根节点的记录查找时间越快**，只要找到关键字即可确定记录的存在；
   3. B+树的**磁盘读写代价更低** ，B+树的内部结点并没有指向关键字具体信息的指针。因此其内部结点相对 B 树更小。
6. **innodb_page_size**是一个初始化数据库实例的参数，在目前的版本中（>=5.7.6），可以选择的值有4096, 8192, 16384, 32768, 65536。默认是16KB。
   一般越小，内存划分粒度越大，使用率越高，但是会有其他问题，就是限制了索引字段还有整行的大小。innodb引擎读取内存还有更新都是一页一页更新的，这个innodb_page_size决定了，一个基本页的大小。常用B+Tree索引，**B+树是为磁盘及其他存储辅助设备而设计一种平衡查找树**（不是二叉树）。B+树中，所有记录的节点按大小顺序存放在同一层的叶子节点中，各叶子节点用指针进行连接。MySQL将每个叶子节点的大小设置为一个页的整数倍，利用**磁盘的预读机制**，能有效**减少磁盘I/O次数**，提高查询效率。 如果一个行数据，超过了一页的一半，那么一个页只能容纳一条记录，这样B+Tree在不理想的情况下就变成了双向链表。

## Hash索引

哈希索引能以 O(1) 时间进行查找，但是失去了有序性：

- 无法用于排序与分组；
- 只支持精确查找，无法用于部分查找和范围查找。

## 前缀索引

对于 BLOB、TEXT 和 VARCHAR 类型的列，必须使用前缀索引，只索引开始的部分字符。

前缀长度的选取需要根据索引选择性来确定。

## 如何为表字段添加索引

1、主键索引 ： ALTER TABLE `table_name` ADD PRIMARY KEY ( `column` ) 

2、唯一索引 ： ALTER TABLE `table_name` ADD UNIQUE ( `column` ) 

3、普通索引 ： ALTER TABLE `table_name` ADD INDEX index_name ( `column` )

4、全文索引 ： ALTER TABLE `table_name` ADD FULLTEXT ( `column`) 

也可以实现  create index  index_name on `table_name` ( `column`)

### 适合创建索引的情况

- 主键自动建立唯一索引；
- 频繁作为查询条件的字段应该创建索引;
- 查询中与其它表关联的字段，外键关系建立索引；
- 单键/组合索引的选择问题， 组合索引性价比更高；
- 查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度；
- 查询中统计或者分组字段。

### 不适合创建索引的情况

- 表记录太少
- 经常增删改的表或者字段
- Where 条件里用不到的字段不创建索引
- 过滤性不好的不适合建索引

## 小问题

1. 什么是主键索引？

   主键索引是指唯一索引，创建表中的主键就会建立主键索引，也称聚簇索引。

2. 什么是联合索引？

   对表里的多列同时建立的索引，根据创建联合索引的顺序，以最左原则进行检索。

3. 什么是覆盖索引？

   根据辅助索引就可以直接查询到结果，而不需要回表到聚簇索引中进行再次查询，可以减少IO次数。

4. 什么是索引条件下推？

   MySQL 5.6引入了索引下推优化，可以在索引遍历过程中，对索引中包含的字段先做判断，过滤掉不符合条件的记录，减少回表字数。

MySQL 5.7 版本后，可以通过查询 sys 库的 `schema_redundant_indexes` 表来查看冗余索引

## ACID

- **原子性**（Atomicity）
  - 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
- **一致性**（Consistency）
  - 执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的；
- **隔离性**（Isolation）
  - 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
- **持久性**（Durability）
  - 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。
  - `MySQL` 使用 `redo log` 来保证事务的持久性

# 并发一致性问题

- **脏读（Dirty read）:**当一个事务正在访问数据并且对数据进行修改，而这种修改还没有提交到数据库中，这时另外一个事务也访问了这个数据。因为这个数据是还没有提交的数据，那么另外一个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。
- **丢失修改（Lost to modify）:** 指一个事务读取一个数据时，另一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内修改的数据结果就丢失了，因此称为丢失修改。 例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失。
- **不可重复读（Unrepeatableread）:** 指在一个事务内多次读同一数据。在这个事务还没有结束时，另一个事务也访问该数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改导致第一个事务两次读取的数据可能不太一样。这就发生了在一个事务内两次读到的数据是不一样的情况，因此称为不可重复读。
- **幻读（Phantom read）:** 幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

**不可重复读和幻读区别：**

不可重复读的重点是修改比如多次读取一条记录发现其中某些列的值被修改，幻读的重点在于**新增或者删除**比如多次读取一条记录，发现记录增多或减少了。

## 四大隔离级别

**SQL 标准定义了四个隔离级别：**

- **READ-UNCOMMITTED(读取未提交)：** 最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读。
- **READ-COMMITTED(读取已提交)：** 允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生。
- **REPEATABLE-READ(可重复读)：** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。
- **SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。

```
# 查看隔离级别
SELECT @@tx_isolation;
```

MySQL默认隔离级别是是可重复读，但也是使用了Next-Key Lock算法避免幻读。

**InnoDB存储引擎的锁的算法有三种：**

- Record lock：单个行记录上的锁
- Gap lock：间隙锁，锁定一个范围，不包括记录本身
- Next-key lock：record+gap 锁定一个范围，包含记录本身

# 参见问题

```mysql
# 查看引擎
show engines;
# 查看最大连接数量
show variables like '%max_connections%';
```

## 死锁

死锁是指两个或者多个事务在同一资源上相互占用，并请求锁定对方占用的资源。**InnoDB是将持有最少行级排它锁的事务进行回滚。**

## MyISAM 和InnoDB区别

5.5之前MyISAM是MySQL的默认数据库引擎，支持全文索引，压缩，空间函数等，但是不支持行级锁和事务，而且最大的缺陷是崩溃了就不能完全恢复。5.5之后InnoDB称为默认的存储引擎，

1. 是否支持行级锁
2. 是否支持事务和崩溃后的安全恢复
3. 是否支持外键
4. 是否支持MVCC
5. 虽然也有.frm 文件来存放 表结构定义相关的元数据，但是表数据和索引数据是存放在一起的。

## 查询缓存

执行查询语句的时候，会先查询缓存。开启设置`query_cache_type`和`query_cache_size`这两个参数。首先缓存虽然会提升查询性能，但是会带来额外的开销。发生了写操作后，缓存就失效了。

> MySQL 8.0 版本后移除

```mysql
select SQL_NO_CACHE * from XXX ; 不带cache
```

## 大表优化

1. 限定数据的范围
2. 读写分离
3. 垂直分区
   1. 减少读取的Block数，减少IO次数，简化表结构，易于维护。
   2. 主键冗余，让事务更复杂
4. 水平分区
   1. 支持非常大的数据量
   2. 数据库分片方案
      1. 客户端代理：分片逻辑在应用端，jar包，通过修改或者封装JDBC层来实现。Sharding-JDBC、TDDL。
      2. 中间件代理：在应用和数据中间加个代理层，分片逻辑统一维护在中间件服务中。Mycat、Altas、DDB。

## 乐观锁与悲观锁



## 多版本并发控制MVCC(multiversion concurrency control)

在MVCC中事务的修改操作会为数据行新增一个版本快照。MVCC规定只能读取到已经提交的快照。

InnoDB是一个多版本的存储引擎：它保存关于旧版本的修改行信息，以支持并发和回滚等事务性特性。这些信息存储在表空间中的数据结构中，称为回滚段。InnoDB使用回滚段中的信息来执行事务回滚所需的撤销操作。它还使用这些信息来构建行的早期版本，以便进行一致的读取。

在内部，InnoDB为存储在数据库中的每一行添加了三个字段。一个6字节的DB_TRX_ID字段表示插入或更新行的最后一个事务的事务标识符。此外，删除操作在内部被视为更新，行中的特殊位被设置为将其标记为已删除。每一行还包含一个名为roll pointer的7字节DB_ROLL_PTR字段。滚动指针指向写入回滚段的undo logs记录。如果更新了行，那么undo日志记录包含了在更新行之前重建该行内容所需的信息。一个6字节的DB_ROW_ID字段包含一个行ID，这个行ID随着插入新行而单调增加。如果InnoDB自动生成一个聚集索引，那么该索引将包含行ID值。否则，DB_ROW_ID列不会出现在任何索引中。

### Undo 日志

回滚段中的undo logs分为插入和更新undo logs。只有在事务回滚时才需要插入undo logs，并且可以在事务提交时立即丢弃。更新undo logs也用于读取一致，但是，只有在没有事务存在的情况下，InnoDB才可以丢弃它们。InnoDB为事务分配了一个快照，在一致的读操作中，这个快照可能需要update undo log中的信息来构建数据库行的早期版本。

### ReadView

MVCC维护一个ReadView结构，主要包含了当前系统未提交的事务列表TRX_IDs{TRX_ID_1, TRX_ID_2, ...}, 还有该列表的最小值和最大值。

##  范式



## 索引以及优化

1. 优化更需要优化的Query
2. 定位优化对象的性能瓶颈
3. 明确优化目标
4. 从Explain入手
5. 多使用profile
6. 永远用小结果集驱动大的结果集； 
7. 尽可能在索引中完成排序； 
8. 只取出自己需要的 Columns；
9. 仅仅使用最有效的过滤条件； 
10. 尽可能避免复杂的 Join 和子查询；

MySQL 的 Query Profiler 是一个使用非常方便的 Query 诊断分析工具，通过该工具可以获取一条 Query 在整个执行过程中多种资源的消耗情况，如 CPU，IO，IPC，SWAP等，以及发生的 PAGE FAULTS，CONTEXT SWITCHE 等等，同时还能得到该 Query 执行过程中 MySQL 所调用的各个函数在源文件中的位 置。下面我们看看 Query Profiler 的具体用法。 

## 日志

- Error log
  - 错误日志记录了 MyQL Server 运行过程中所有较为严重的警告和错误信息，以及 MySQL Server 每次启动和关闭的详细信息。
- query log
- slow query log
- **undo log**
  - 回滚和多版本并发控制(`MVCC`)
  - 存储着修改之前的数据，如果有`update`，则存储对应的`delete`
- **binlog**
  - 复制和恢复数据
  - 主从服务器需要保持数据的一致性，通过`binlog`来同步数据
- InnoDB的**redo log**
  - `redo log`记录的是数据的**物理变化**，`binlog`记录的是数据的**逻辑变化**
  - 不会存储着**历史**所有数据的变更，**文件的内容会被覆盖的**。

## 慢查询

MySQL的慢查询日志是用来记录响应时间超过阈值的语句，具体指运行时间超过`long_query_time`值的SQL，则会被记录到慢查询日志中。默认是不启动，如果有调优需要才启用该参数。

```
slow_query_log    ：是否开启慢查询日志，1表示开启，0表示关闭。
long_query_time ：慢查询阈值，当查询时间多于设定的阈值时，记录日志。
log_output：日志存储方式。log_output='FILE'表示将日志存入文件，默认值是'FILE'。log_output='TABLE'表示将日志存入数据库，这样日志信息就会被写入到mysql.slow_log表中。
```

具体配置如下：

```
show variables likes '%slow_query_log%';
set global slow_query_log = 1;
```

MySQL重启后则会失效。如果要永久生效，就必须修改配置文件my.cnf。 	



## Mysql文件排序





# 关系型数据库设计理论

## 函数依赖

- 对于 A->B，B->C，则 A->C 是一个传递函数依赖。



## 安装

```
yum install mysql-community-server
systemctl enable mysqld
systemctl daemon-reload
systemctl start mysqld
grep 'temporary password' /var/log/mysqld.log
mysql -uroot -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

## Hint

**强制索引 FORCE INDEX** 
SELECT * FROM TABLE1 FORCE INDEX (FIELD1) …
以上的SQL语句只使用建立在FIELD1上的索引，而不使用其它字段上的索引。
**忽略索引 IGNORE INDEX** 
SELECT * FROM TABLE1 IGNORE INDEX (FIELD1, FIELD2) …
在上面的SQL语句中，TABLE1表中FIELD1和FIELD2上的索引不被使用。 
**关闭查询缓冲 SQL_NO_CACHE** 
SELECT SQL_NO_CACHE field1, field2 FROM TABLE1;
有一些SQL语句需要实时地查询数据，或者并不经常使用（可能一天就执行一两次）,这样就需要把缓冲关了,不管这条SQL语句是否被执行过，服务器都不会在缓冲区中查找，每次都会执行它。
**强制查询缓冲 SQL_CACHE**
SELECT SQL_CALHE * FROM TABLE1;
如果在my.ini中的query_cache_type设成2，这样只有在使用了SQL_CACHE后，才使用查询缓冲。

**优先操作 HIGH_PRIORITY**
HIGH_PRIORITY可以使用在select和insert操作中，让MYSQL知道，这个操作优先进行。
SELECT HIGH_PRIORITY * FROM TABLE1;
**滞后操作 LOW_PRIORITY**
LOW_PRIORITY可以使用在insert和update操作中，让mysql知道，这个操作滞后。
update LOW_PRIORITY table1 set field1= where field1= …
**延时插入 INSERT DELAYED**
INSERT DELAYED INTO table1 set field1= …
INSERT DELAYED INTO，是客户端提交数据给MySQL，MySQL返回OK状态给客户端。而这是并不是已经将数据插入表，而是存储在内存里面等待排队。当mysql有空余时，再插入。另一个重要的好处是，来自许多客户端的插入被集中在一起，并被编写入一个块。这比执行许多独立的插入要快很多。坏处是，不能返回自动递增的ID，以及系统崩溃时，MySQL还没有来得及插入数据的话，这些数据将会丢失。

**强制连接顺序 STRAIGHT_JOIN**
SELECT TABLE1.FIELD1, TABLE2.FIELD2 FROM TABLE1 STRAIGHT_JOIN TABLE2 WHERE …
由上面的SQL语句可知，通过STRAIGHT_JOIN强迫MySQL按TABLE1、TABLE2的顺序连接表。如果你认为按自己的顺序比MySQL推荐的顺序进行连接的效率高的话，就可以通过STRAIGHT_JOIN来确定连接顺序。
**强制使用临时表 SQL_BUFFER_RESULT**
SELECT SQL_BUFFER_RESULT * FROM TABLE1 WHERE … 
当我们查询的结果集中的数据比较多时，可以通过SQL_BUFFER_RESULT.选项强制将结果集放到临时表中，这样就可以很快地释放MySQL的表锁（这样其它的SQL语句就可以对这些记录进行查询了），并且可以长时间地为客户端提供大记录集。
**分组使用临时表 SQL_BIG_RESULT和SQL_SMALL_RESULT**
SELECT SQL_BUFFER_RESULT FIELD1, COUNT(*) FROM TABLE1 GROUP BY FIELD1;
一般用于分组或DISTINCT关键字，这个选项通知MySQL，如果有必要，就将查询结果放到临时表中，甚至在临时表中进行排序。SQL_SMALL_RESULT比起SQL_BIG_RESULT差不多，很少使用。

**参考：** 

[1] https://en.wikipedia.org/wiki/B%2B_tree

[2] https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html

[3] https://github.com/Snailclimb/JavaGuide

[4] https://github.com/CyC2018/CS-Notes/

