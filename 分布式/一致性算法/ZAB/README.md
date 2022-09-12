# 概念
Zab协议的全称是 Zookeeper Atomic Broadcast （Zookeeper原子广播）。Zookeeper 是通过Zab协议来保证分布式事务的最终一致性。
- Zab协议是为分布式协调服务Zookeeper专门设计的一种支持崩溃恢复的原子广播协议 ，是Zookeeper保证数据一致性的核心算法。Zab借鉴了Paxos算法，但又不像Paxos那样，是一种通用的分布式一致性算法。它是特别为Zookeeper设计的支持崩溃恢复的原子广播协议。
- 在Zookeeper中主要依赖Zab协议来实现数据一致性，基于该协议，zk实现了一种主备模型（即Leader和Follower模型）的系统架构来保证集群中各个副本之间数据的一致性及顺序性。这里的主备系统架构模型，就是指只有一台客户端（Leader）负责处理外部的写事务请求，然后Leader客户端将数据同步到其他Follower节点。

Zookeeper客户端会随机的链接到Zookeeper集群中的一个节点，如果是读请求，就直接从当前节点中读取数据；如果是写请求，那么节点就会向Leader提交事务，Leader接收到事务提交，会广播该事务，只要超过半数节点写入成功，该事务就会被提交。

ZAB协议规定了如果一个事务在一台机器上被处理(commit)成功，那么应该在所有的机器上都被处理成功，哪怕机器出现 故障崩溃。

<br>

# ZAB协议三阶段
- 发现（Discovery）：选举Leader过程。
- 同步（Synchronization）：选举出新的Leader后，Follwer或者Observer从Leader同步最新的数据。
- 广播：同步完成后，就可以接收客户端新的事务请求，并进行消息广播，实现数据在集群节点的副本存储。

<br>

# Leader选举过程
## 初次选举
 三个ZK节点，都没有数据，对应的myid分别为1,2,3
- 第一步，启动myid为1的节点，此时zxid为0，此时没法选举出主节点
- 第二步，启动myid为2的节点，此时zxid为0，此时2节点选举为主节点
- 第三步，启动myid为3的节点，因为已经有主节点，节点3加入，节点2仍然是主节点

## 宕机选举
- 场景说明<br>
3台机器，此时server2为主，并且sever2宕机
- 选举流程
    1. 变更状态<br>
    当leader宕机后，其他节点的状态变味LOOKING，每个server发出一个投自己的票的投票，生成投票信息（myid, zxid），假定：server1为（1,123），server3为（2,122），server1发给server3，server3发给server1
    2. 接收投票
    3. 投票处理<br>
    server3收到server1，因为server1的123比122大，所以server3修改自己的投票为（1,123）然后发给server1，server1收到server3的投票，因为123大于122，因此不改变自己的投票。
    4. 统计投票<br>
    server3统计：自己收到投票（包括自己投的）中，（1,123）是2票<br>
    server1统计：自己收到投票（包括自己投的）中，（1,123）是2票
    5. 修改服务器状态<br>
    server3：选出的leader是1，而自己是3，因此自己进入Followering状态，即Follower角色<br>
    server1，选出的leader是1，自己就是1，因此自己进入LEADING状态，即自己是Leader角色

宕机选举完毕后，现leader和follower有可能会有一定的差异，需要进行同步

<br>

# 影响成为Leader的因素
## 数据新旧程度
只有拥有最新数据的节点才有机会成为Leader，通过事物id（zxid）的大小表示数据的新旧，越大代表数据越新

## myid
集群启动前，会在data目录下配置myid文件，内容代表ZK服务节点的编号，当ZK服务器节点数据都保持最新数据时，myid中数字越大的就会被选举为Leader，当集群中已有Leader时，新加入的节点不会影响原来的集群

## 投票数量
只有获得过半投票才能选举为Leader，多半即：n/2 + 1，其中n为集群节点数，优先用数据的新旧，其次用myid的大小
> **zxid的构成**
>1. 主进程周期<br>
也叫epoch，每进行一次选举，主进程的周期加一，
zxid总共由64位组成，其高32位代表主进程周期，
比较数据新旧的时候，先比较epoch的大小
>2. 事务单调递增的计数器
>3. zxid低32位表示，选举完成后，从0开始

<br>

# 广播流程
- 集群选举完成，并且完成数据同步后，即可开始对外服务，接收读写请求
- 当Leader接收到客户端新的事务请求后，会生成对应的事务proposal，并根据zxid的顺序所有的follower发送提案，即proposal
- 当follower收到leader的事务proposal时，根据接收的先后顺序处理这些proposal，即如果先后收到1,2,3条，则如果处理完了3条，则代表1,2两条一定已经处理成功
- 当Leader收到follower针对某个事务proposal过半的ack后，则发起事务提交，重新发起一个commit的proposal
- follower收到commit的proposal后，记录事务提交，并把数据更新到内存数据库
- 补充说明<br>
由于只有过半的机器给出反馈，则可能存在某时刻某些节点数据不是最新的
业务上如果需要读取到的数据时最新的，则可以在读取之前，调用sync方法进行数据同步

<br>

# 崩溃恢复
- Leader 在复制数据给所有 Follwer 之后，还没来得及收到Follower的ack返回就崩溃，怎么办？ —— 此时没有节点commit，在重新进行leader选举后，这条数据被删除。
- Leader 在收到 ack 并提交了自己，同时发送了部分 commit 出去之后崩溃怎么办？在重新进行leader选举后，新的leader将这条数据广播给其他节点的事务。

ZAB 定义了 2 个原则：
- ZAB 协议确保丢弃那些只在 Leader 提出/复制，但没有提交的事务。
- ZAB 协议确保那些已经在 Leader 提交的事务最终会被所有服务器提交。
>这个“已经被leader提交的proposal”的 正确理解：只要leader提出的proposal被过半的节点接收到(包含了leader自己)，即完成ACK，那么就算是被leader提交成功了(不论leader有没发出commit命令)，如果leader根本没发出commit命令就挂了，那么后续被选举的新leader（zxid最大的那个）,会首先判断自身未Commit的消息是否存在于大多数服务器中从而决定是否要将其Commit。

同时：<br>
Leader 选举算法能够保证新选举出来的 Leader 服务器拥有集群中所有机器 ZXID 最大。两个选择+选举算法保证了数据一致性。

# 参考资料
[ZAB选举算法](https://blog.csdn.net/iflink/article/details/122389618)

[ZAB算法概览](https://www.jianshu.com/p/af69f59565f0)

[ZAB算法核心流程](https://www.jianshu.com/p/672ce06b8b8d)