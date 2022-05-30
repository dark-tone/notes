# DDL与DML
DDL(data definition language)，表结构定义语句，CREATE、ALTER、DROP等更改表结构的语句。

DML(data manipulation language)，数据的增删查改语句。

# Online DDL
在MySQL5.5以及之前的版本，通常更改数据表结构操作(DDL)会阻塞对表数据的增删改操作(DML)

MySQL5.6提供Online DDL之后可支持DDL与DML操作同时执行，也就是降低了DDL期间对业务延迟带来的影响

## 特点
1. DDL操作可与应用的DML操作并发执行，改进在繁忙生产环境的响应和可用性，因为对于这种业务系统几分钟或几小时不可用是难以忍受的
2. 可以通过调整DDL的lock模式来平衡与DML操作的性能问题
3. Online DDL使用的是In-place方式，相较于table-copy方式能更少的使用I/O资源，在DDL期间也能有较高的吞吐量
4. In-place相较于table-copy还有一个优点是：table-copy读取数据多，频繁的使用buffer pool导致有效缓存数据被调出，影响缓存击中率 降低效率

# 原理
1. 拿MDL写锁
2. 降级成MDL读锁
3. 真正做DDL
4. 升级成MDL写锁
5. 释放MDL锁

1、2、4、5如果没有锁冲突，执行时间非常短。第3步占用了DDL绝大部分时间，这期间这个表可以正常读写数据，是因此称为“online”。

# inplace和copy方式的区别
1. ALGORITHM=COPY是MySQL5.5以及之前的方式
2. ALGORITHM=INPLACE是MySQL5.6引入的方式
3. COPY算法，由service层创建一个临时表用于copy数据，然后用新表替换旧表
4. INPLACE算法，“原位替换” 其实主要是指在InnoDB内部完成的DDL操作，在InnoDB内部创建临时文件。整个 DDL 过程都在 InnoDB 内部完成。对于 server 层来说，没有把数据挪动到临时表，是一个“原地”操作，这就是“inplace”名称的来源。
5. 因此对于INPLACE其实分为非重建表和重建表两类方式，非重建表方式直接在原表基础上更新，效率最高；重建表同样需要copy数据(比如新增字段)

# 参考资料
《MySQL实战45讲》

[浅谈MySQL Online DDL](https://zhuanlan.zhihu.com/p/355217200)