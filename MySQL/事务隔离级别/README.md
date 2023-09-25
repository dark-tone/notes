# 事务隔离级别
## 事务四特性
ACID（Atomicity、Consistency、Isolation、Durability，即原子性、一致性、隔离性、持久性）

## 常见并发事务问题
- **脏读**<br>
B事务读取了A事务未提交的更改。

- **不可重复读**<br>
B事务读取了两次数据，在这两次的读取过程中A事务**修改**了数据，B事务的这两次读取出来的数据不一样。

- **幻读**<br>
B事务读取了两次数据，在这两次的读取过程中A事务**添加**了数据，B事务的这两次读取出来的集合不一样（当前读，insert的情况）。

## 隔离级别
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/MySQL/imgs/1.jpg" weight="710" height="114">

- 读未提交（一个事务还没提交时，它做的变更就能被别的事务看到）
- 读已提交（RC，一个事务提交之后，它做的变更才会被其他事务看到，Oracle默认级别）
    > RC 隔离级别不支持 statement 格式的bin log，因为该格式的复制，会导致主从数据的不一致；只能使用 mixed 或者 row 格式的bin log。
- 可重复读（RR，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。）
    >MySQL Innodb RR隔离级别下能够解决幻读的场景及原因：<br>
1.快照读，由MVCC解决幻读。<br>
2.当前读，使用next-key lock，避免该范围内的幻读。依赖索引进行，如果没有索引则会把整个表都锁住。<br>
- 串行化（顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行）

### RR级别与幻读
RR级别依然可能发生幻读，比如有两个事务A与B。

1. 事务A先用普通select查询语句（此时会使用MVCC，因为是快照读）。
2. 事务B使用insert插入数据。
3. 事务A使用update更新数据。
4. 事务A再查询，此时会发现事务B之前插入的数据。

解决办法是事务A的首次查询，使用select ... for update，此时即可使用当前读来锁住需要更新的行。

### next-key lock
next-key lock = 行锁（reocrd lock） + 间隙锁（gay lock）

间隙锁的锁定范围为**左开右闭**。注意当使用**唯一索引**来搜索唯一行的语句时，不需要间隙锁定。

## undo log
为逻辑日志，可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。（会形成一个版本链）

### 存储方式
undo log默认存放在共享表空间中(ibdata1)。如果开启了 innodb_file_per_table ，将放在每个表的.ibd文件中。

默认rollback segment全部写在一个文件中，但可以通过设置变量 innodb_undo_tablespaces 平均分配到多少个文件中。该变量默认值为0，即全部写入一个表空间文件。

### 释放时间
在事务提交的时候，会将该事务对应的undo log放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间。

## 多版本并发控制（MVCC）
MVCC 是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问。应用于 Read Commited 和 Repeatable read 两个事务隔离级别。

### 快照读和当前读
- **当前读**<br>
像 select lock in share mode (共享锁), select for update; update; insert; delete (排他锁)这些操作都是一种当前读，为什么叫当前读？就是它读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。

- **快照读**<br>
像不加锁的 select 操作就是快照读，即不加锁的非阻塞读；快照读的前提是隔离级别不是串行级别，串行级别下的快照读会退化成当前读；之所以出现快照读的情况，是基于提高并发性能的考虑，快照读的实现是基于多版本并发控制，即 MVCC ,可以认为 MVCC 是行锁的一个变种，但它在很多情况下，避免了加锁操作，降低了开销；既然是基于多版本，即快照读可能读到的并不一定是数据的最新版本，而有可能是之前的历史版本

### 数据的隐藏字段
数据库插入数据后，会额外加入两个隐藏字段trx_id 和 roll_pointer：
- trx_id：当前事务ID。
- roll_pointer：指向 undo log 的指针。

### Read View(读视图)
Read View就是事务进行快照读操作的时候生产的读视图(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID。

这里有四个概念：
- creator_trx_id，当前事务ID。
- m_ids，生成 readView 时还活跃的事务ID集合，也就是已经启动但是还未提交的事务ID列表。
- min_trx_id，当前活跃ID之中的最小值。
- max_trx_id，生成 readView 时 InnoDB 将分配给下一个事务的 ID 的值（事务 ID 是递增分配的，越后面申请的事务ID越大）

对于可见版本的判断是从最新版本开始沿着版本链逐渐寻找老的版本，如果遇到符合条件的版本就返回。判断条件如下：
- 如果当前数据版本的 trx_id == creator_trx_id 说明修改这条数据的事务就是当前事务，所以可见。
- 如果当前数据版本的 trx_id < min_trx_id，说明修改这条数据的事务在当前事务生成 readView 的时候已提交，所以可见。
- 如果当前数据版本的 trx_id 大小在 min_trx_id 和 max_trx_id 之间，此时 trx_id 若在 m_ids 中，说明修改这条数据的事务此时还未提交，所以不可见，若不在 m_ids 中，表明事务已经提交，可见。
- 如果当前数据版本的 trx_id >= max_trx_id，说明修改这条数据的事务在当前事务生成 readView 的时候还未启动，所以不可见(结合事务ID递增来看)。

### RC与RR级别的MVCC区别
可重复读和读已提交的 MVCC 判断版本的过程是一模一样的，唯一的差别在生成 readView 上。<br>
上面的读已提交一个事务中的每次查询都会重新生成一个新的 readView ，而可重复读在第一次生成 readView 之后的所有查询都共用同一个 readView 。<br>
也就是说可重复读只会在第一次 select 时候生成一个 readView ，所以一个事务里面不论有几次 select ，其实看到的都是同一个 readView 。

### 如何降低加锁时间
当我们在编写一个事务的时候，加行锁的操作应在不影响业务的情况下，尽可能地靠近 commit 语句，这样单行记录的行锁时间才会更短，TPS 会更高。

# 参考文章
《MySQL实战45讲》

[Innodb中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)

[一个 MVCC 和面试官大战 30 回合](https://zhuanlan.zhihu.com/p/383842414)

[正确的理解MySQL的MVCC及实现原理](https://blog.csdn.net/SnailMann/article/details/94724197)

[大促场景下库存更新 SQL 优化](https://juejin.cn/post/7266302333634215976)