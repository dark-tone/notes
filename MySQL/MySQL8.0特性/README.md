# 新特性
### 降序索引
```
create table test (c1 int,c2 int,index idx_c1_c2(c1,c2 desc))
```
Backward index scan：反向索引扫描。<br>
多个字段，排序方向不同时，使用对应的降序索引可以避免filesort。

### group by不再隐式排序
如果要添加排序的话，需要显示增加，比如 select count(*),age from t5 group by age order by age;

### 隐藏索引功能
可用于测试在线索引的性能。

### 死锁检查控制
可以使用一个新的动态变量 innodb_deadlock_detect 来禁用死锁检测。在高并发系统上，当多个线程等待同一个锁时，死锁检测会导致速度变慢。有时，禁用死锁检测并在发生死锁时依靠 innodb_lock_wait_timeout 设置进行事务回滚可能更有效。

### undo文件不再使用系统表空间
默认创建2个UNDO表空间，不再使用系统表空间。(原来的undo文件是放在ibd文件里的，如果独立出来需要配置)。

### MyISAM系统表全部换成InnoDB表
将系统表(mysql)和数据字典表全部改为InnoDB存储引擎，默认的MySQL实例将不包含MyISAM表，除非手动创建MyISAM表。

### DDL原子化
InnoDB表的DDL支持事务完整性，要么成功要么回滚，将DDL操作回滚日志写入到data dictionary 数据字典表 mysql.innodb_ddl_log 中用于回滚操作，该表是隐藏的表，通过show tables无法看到。通过设置参数，可将ddl操作日志打印输出到mysql错误日志中。

### 移除查询缓存

### JSON特性增强

### 基于竞争感知的事务调度
MySQL 在 8.0.3 版本引入了新的事务调度算法，基于竞争感知的事务调度，Contention-Aware Transaction Scheduling，简称CATS。在CATS算法之前，MySQL使用FIFO算法，先到的事务先获得锁，如果发生锁等待，则按照FIFO算法进行排队。CATS相比FIFO更加复杂，也更加聪明，在高负载、高争用的场景下，性能提升显著。

### 安全及账号管理

### 公用表表达式CTE

### 默认字符集由latin1变为utf8mb4

### 参数修改持久化
MySQL 8.0版本支持在线修改全局参数并持久化，通过加上PERSIST关键字，可以将修改的参数持久化到新的配置文件（mysqld-auto.cnf）中，重启MySQL时，可以从该配置文件获取到最新的配置参数。

### Binlog增强
MySQL 8.0.20 版本增加了binlog日志事务压缩功能，将事务信息使用zstd算法进行压缩，然后再写入binlog日志文件，这种被压缩后的事务信息，在binlog中对应为一个新的event类型，叫做Transaction_payload_event。

### redo优化
mysql8.0一个新特性就是redo log提交的无锁化。在8.0以前，各个用户线程都是通过互斥量竞争，串行的写log buffer，因此能保证lsn的顺序无间隔增长。

mysql8.0通过redo log无锁化，解决了用户线程写redo log时竞争锁带来的性能影响。同时将redo log写文件、redo log刷盘从用户线程中剥离出来，抽成单独的线程，用户线程只负责将redo log写入到log buffer，不再关心redo log的落盘细节，只需等待log_writer线程或log_flusher线程的通知。

# 参考资料

[MySQL 8.0 新特性解读(上)](https://mp.weixin.qq.com/s/AafVRpOrCTKA7Zg0E3eixQ)

[MySQL 8.0 新特性解读(下)](https://opensource.actionsky.com/20230322-mysql/)

[MySQL 8.0 新特性](https://www.cnblogs.com/bantiaoxianyu/p/16485658.html)

[MySQL8.0之降序索引](https://cloud.tencent.com/developer/article/1832961)