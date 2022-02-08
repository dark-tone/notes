# 原理
主库启动io线程读取binlog日志并发送到从库，从库启动io线程读取发来的binlog日志，然后解析并写入relaylog，另有一个sql线程读取relaylog并执行相应的sql语句。

# 简要步骤
1. 安装好配置从库的数据库，配置好log-bin和server-id参数
2. 无需配置主库my.cnf，主库的log-bin和server-id参数默认就是配置好的。
3. 登录主库，增加从库连接同步的账户，例如：rep，并授权replication同步的权限
4. 使用mysqldump命令带-x和--master-data=2的命令及参数全备数据，把它恢复到从库
5. 从库执行CHANGE MASTER TO....语句，需要binlog文件及对应点（因为--master-data=2已经带了）
6. 从库开启同步开关，start slave
7. 从库show slave status，检查同步状态，并在主库更新测试

# 同步方式
## 异步复制
主库事务的提交，与同步的binlog复制是异步进行的
## 全同步复制
主库事务的提交，需从库确认提交后再提交
## 半同步复制
主库事务的提交，在从库的relaylog写入完毕后发回确认信息再提交

# 参考资料
[mysql主从复制 主主复制](https://segmentfault.com/a/1190000009724090)