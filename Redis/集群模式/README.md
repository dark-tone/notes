# 优点
分摊流量，相当于横向拓展。
> 另，数据量过大的话，单机 RDB 持久化时，Redis 会 fork 子进程来完成，fork 操作的用时和 Redis 的数据量是正相关的，而 fork 在执行时会阻塞主线程。数据量越大，fork 操作造成的主线程阻塞的时间越长。所以，在使用 RDB 对 25GB 的数据进行持久化时，数据量较大，后台运行的子进程在 fork 创建时阻塞了主线程，于是就导致 Redis 响应变慢了。

# 原理
Redis Cluster 方案采用哈希槽（Hash Slot），来处理数据和实例之间的映射关系。在 Redis Cluster 方案中，一个切片集群共有 16384 个哈希槽，这些哈希槽类似于数据分区，每个键值对都会根据它的 key，被映射到一个哈希槽中。

具体的映射过程分为两大步：首先根据键值对的 key，按照CRC16 算法计算一个 16 bit 的值；然后，再用这个 16bit 值对 16384 取模，得到 0~16383 范围内的模数，每个模数代表一个相应编号的哈希槽。

在部署 Redis Cluster 方案时，可以使用 cluster create 命令创建集群，此时，Redis 会自动把这些槽平均分布在集群实例上。例如，如果集群中有 N 个实例，那么，每个实例上的槽个数为 16384/N 个。我们也可以使用 cluster meet 命令手动建立实例间的连接，形成集群，再使用cluster addslots 命令，指定每个实例上的哈希槽个数。

# 手动部署例子
1. 每个redis节点的redis.conf加上集群模式所需的配置并启动，如：
```
port 8000
daemonize yes
cluster-enabled yes  // 开启集群模式
cluster-config-file nodes.conf  // 用于存储节点信息的日志文件
cluster-node-timeout 5000
```

2. 构建集群网络
```
// 可以使用cluster meet命令
cluster meet  ip  port

// 亦可使用cluster create命令（注意使用create命令则无需执行创建slot的命令）
redis-cli --cluster create --cluster-replicas 1 192.168.144.79:6379 192.168.144.79:6380 192.168.144.80:6379 192.168.144.80:6380 192.168.144.100:6379 192.168.144.100:6380
```

3. 创建slot
```
// 可以使用cluster addslots命令
```

# 集群模式的高可用
在集群模式中，多个master之间可以充当哨兵的功能，因此无需额外创建sentinel，用 <code>cluster create</code> 命令可以指定从服务器数量，或者手动使用 <code>replicate</code> 命令指定每个master服务器的slave服务器。


# 什么是一致性哈希
知道了通过传统哈希算法来实现对节点的负载均衡的弊端，我们就需要进一步了解什么是一致性哈希。

我们上面提过哈希算法是对master实例数量来取模，而一致性哈希则是对2^32取模，也就是值的范围在[0, 2^32 -1]。一致性哈希将其范围抽象成了一个圆环，使用CRC16算法计算出来的哈希值会落到圆环上的某个地方。
> 例，实例为3，普通的hash算法为：xx % 3，这样当增加一个新实例时需要大面积重新hash。而一致性hash取模时为一个环，只需要移动受影响的slot即可。


# 参考资料
《Redis设计与实现》

[一文读懂 Redis 集群](https://cloud.tencent.com/developer/article/1592432)

[分布式关键技术之一致性哈希和虚拟节点原理介绍](https://baijiahao.baidu.com/s?id=1734276607215305549&wfr=spider&for=pc)