# 哨兵模式
监控主从服务器是否正常工作，并在主服务器出现故障时切换主服务器的功能。

## 示意图
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/16-1.jpg" weight="400" height="290">

server1：主服务器<br>
server2 ~ server4：从服务器

## 实现
#### 1. 配置主从服务器
配置哨兵模式的前提是有主从服务器（使用replicaof配置）
#### 2. 设置哨兵实例
修改和redis.conf同级目录下的sentinel.conf文件
```
# 哨兵的端口号
port 26379

# 设定密码认证
requirepass 123456

# 配置哨兵的监控参数
# 格式：sentinel monitor <master-name> <ip> <redis-port> <quorum>
# master-name是为这个被监控的master起的名字
# ip是被监控的master的IP或主机名。Docker容器之间可以使用容器名访问
# redis-port是被监控节点所监听的端口号
# quorom设定了当几个哨兵判定这个节点失效后，才认为这个节点真的失效了
sentinel monitor local-master 127.0.0.1 6379 2

# 连接主节点的密码
# 格式：sentinel auth-pass <master-name> <password>
sentinel auth-pass local-master 123456

# master在连续多长时间无法响应PING指令后，就会主观判定节点下线，默认是30秒
# 格式：sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds local-master 30000
```
启动哨兵实例：
```
src/redis-sentinel sentinel.conf 
```

## 原理
哨兵启动后会与要监控的主数据库建立两条连接<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/16-2.jpg" weight="677" height="161">

和主数据库连接建立完成后，哨兵会使用连接2发送如下命令:
1. 每10秒钟哨兵会向主数据库和从数据库发送INFO 命令。
2. 每2秒钟哨兵会向主数据库和从数据的_sentinel_:hello频道发送自己的消息。
3. 每1秒钟哨兵会向主数据、从数据库和其他哨兵节点发送PING命令。

&emsp;&emsp;首先,发送INFO命令会返回当前数据库的相关信息(运行id，从数据库信息等)从而实现新节点的自动发现，前面提到的配置哨兵时只需要监控Redis主数据库即可，因为哨兵可以借助INFO命令来获取所有的从数据库信息(slave),进而和这两个从数据库分别建立两个连接。在此之后哨兵会每个10秒钟向已知的主从数据库发送INFO命令来获取信息更新并进行相应的操作。

&emsp;&emsp;接下来哨兵向主从数据库的_sentinel_:hello 频道发送信息来与同样监控该数据库的哨兵分享自己的信息。发送信息内容为:
```
<哨兵的地址>，<哨兵的端口>，<哨兵的运行ID>，<哨兵的配置版本>，
<主数据库的名字>，<主数据库的地址>，<主数据库的端口>，<主数据库的配置版本>
```
&emsp;&emsp;哨兵通过监听的_sentinel_:hello频道接收到其他哨兵发送的消息后会判断哨兵是不是新发现的哨兵，如果是则将其加入已发现的哨兵列表中并创建一个到其的连接(哨兵与哨兵只会创建用来发送PING命令的连接，不会创建订阅频道的连接)。

&emsp;&emsp;实现了自定发现从数据库和其他哨兵节点后，哨兵要做的就是定时监控这些数据和节点运行情况，每隔一定时间向这些节点发送PING命令来监控。间隔时间和down-after-milliseconds选项有关，down-after-milliseconds的值小于1秒时，哨兵会每隔down-after-milliseconds指定的时间发送一次PING命令，当down-after-milliseconds的值大于1秒时，哨兵会每隔1秒发送一次PING命令。

### 主观下线
&emsp;&emsp;当超过down-after-milliseconds指定时间后，如果被PING的数据库或节点仍然未回复，则哨兵认为其主观下线(subjectively down),主观下线表示从当前的哨兵进程看来，该节点已经下线。

### 客观下线
&emsp;&emsp;在主观下线后，如果该节点是主数据库，则哨兵会进一步判断是否需要对其进行故障恢复，哨兵发送SENTINEL is-master-down-by-addr 命令询问其他哨兵节点以了解他们是否也认为该主数据库主观下线，如果达到指定数量时，哨兵会认为其客观下线(objectively down),并选举领头的哨兵节点对主从系统发起故障恢复。这个指定数量就是前面配置的quorum参数。

### 选举领头哨兵
当前哨兵虽然发现了主服务器客观下线，需要故障恢复，但故障恢复需要由领头哨兵来完成。这样来保证同一时间只有一个哨兵来执行故障恢复，选举领头哨兵的过程使用了Raft算法，具体过程如下:
1. 发现主数据库客观下线的哨兵节点(A)向每个哨兵节点发送命令，要求对象选择自己成为领头哨兵。
2. 如果目标哨兵节点没有选过其他人，则会同样将A设置为领头哨兵。
3. 如果A发现有超过半数且超过quorum参数值的哨兵节点同样选择自己成为领头哨兵，则A成功成为领头哨兵。
4. 当有多个哨兵节点同时参选领头哨兵，则会出现没有任何节点当选的可能，此时每个参选节点将等待一个随机事件重新发起参选请求进行下一轮选举，直到选举成功。

### 故障恢复
1. 领头哨兵将从停止服务的主数据库的从数据库中挑选一个来充当新的主数据库。
    > **挑选依据**<br>
a. 删除从服务器列表中的旧数据（下线或断线状态的、最近5秒没有回复领头哨兵INFO命令的、与主服务器连接断开超过down-after-milliseconds * 10毫秒的）    
b. 所有在线的从数据库中，选择优先级最高的从数据库。优先级通过replica-priority参数设置。<br>
c. 优先级相同，则复制的命令偏移量越大(复制越完整)越优先。<br>
d. 如果以上都一样，则选择运行ID较小的从数据库。
2. 选出一个从数据库后，领头哨兵将向从数据库发送SLAVEOF NO ONE命令使其升格为主数据库，而后领头哨兵向其他从数据库发送 SLAVEOF命令来使其成为新主数据库的从数据库.
3. 修改从服务器的复制目标。
4. 将已经停止服务的旧的主数据库更新为新的主数据库的从数据库，使得当其恢复服务时自动以从数据库的身份继续服务。

### 为什么至少需要三个哨兵？
quorum确认odown（客观下线）的最少的哨兵数量，majority授权进行主从切换的最少的哨兵数量（两台就是2，三台也是2，五台就是3），当挂了一台哨兵后，即便可以判定已经客观下线，也无法执行主从切换操作。

## 文章参考
《Redis设计与实现》

[Redis哨兵模式详解](https://cloud.tencent.com/developer/article/1409270)