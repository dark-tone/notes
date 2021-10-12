# 日志模块
## redo log
InnoDB引擎特有的日志。

当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面。

redo log是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/MySQL/imgs/2.png" weight="468" height="351">

**crash-safe**：有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失。这个能力称为 crash-safe。

### redo log buffer
在一个事务的更新过程中，日志是要写多次的。比如下面这个事务：
```
begin;
insert into t1 ...
insert into t2 ...
commit;
```
这个事务要往两个表中插入记录，插入数据的过程中，生成的日志都得先保存起来，但又不能在还没 commit 的时候就直接写到 redo log 文件里。所以，redo log buffer 就是一块内存，用来先存 redo 日志的。也就是说，在执行第一个 insert 的时候，数据的内存被修改了，redo log buffer 也写入了日志。但是，真正把日志写到 redo log 文件（文件名是 ib_logfile+ 数字），是在执行 commit 语句的时候做的。

### redo log的三种状态
- 存在 redo log buffer 中，物理上是在 MySQL 进程内存中；
- 写到磁盘 (write)，但是没有持久化（fsync)，物理上是在文件系统的 page cache 里面，
- 持久化到磁盘，对应的是 hard disk。

事务在执行过程中，生成的 redo log 是要先写到 redo log buffer 的。日志写到 redo log buffer 是很快的，wirte 到 page cache 也差不多，但是持久化到磁盘的速度就慢多了。

### redo log 的写入策略
InnoDB 提供了 innodb_flush_log_at_trx_commit 参数，它有三种可能取值：
- 设置为 0，表示每次事务提交时都只是把 redo log 留在 redo log buffer 中;
- 设置为 1，表示每次事务提交时都将 redo log 直接持久化到磁盘;
- 设置为 2，表示每次事务提交时都只是把 redo log 写到 page cache。

## binlog
归档日志，位于Server层。

### MySQL 怎么知道 binlog 是完整的?
一个事务的 binlog 是有完整格式的：
statement 格式的 binlog，最后会有 COMMIT；row 格式的 binlog，最后会有一个 XID event。

### redo log 和 binlog 是怎么关联起来的?
它们有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log：
- 如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
- 如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。

## 两种日志的不同点
1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

## Update语句的内部流程
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/MySQL/imgs/3.png" weight="480" height="639">

**两阶段提交**：redo log 的写入拆成了两个步骤：prepare 和 commit，这就是"两阶段提交"。redo log 和 binlog 都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。

### 两阶段提交时异常重启的处理
1. 写入 redo log 处于 prepare 阶段之后、写 binlog 之前，发生了崩溃（crash），由于此时 binlog 还没写，redo log 也还没提交，所以崩溃恢复的时候，这个事务会回滚。
2. binlog 写完，redo log 还没 commit 前发生 crash时：
    - 如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；
    - 如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整，如果是，则提交事务，否则，回滚事务。

## WAL技术（Write-Ahead Logging）
先写日志，再写磁盘。

# 参考文章
《MySQL实战45讲》

[MySQL详解](https://blog.csdn.net/weixin_47061482/article/details/115163442)