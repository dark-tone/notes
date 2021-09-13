# 主从复制
执行slaveof命令或者设置slaveof选项（redis5.0后使用replicaof）即可实现主从复制。

## 示意图
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/10-2.jpg" weight="572" height="420">

第一次同步：<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/10-1.webp" weight="544" height="242">

## 部分重同步的实现
### 构成
1. 复制偏移量（offset）
2. 主服务器的复制积压缓冲区
3. 服务器运行ID（runID）

### 复制积压缓冲区
固定长度的先进先出队列。<br>
示意图：<br>
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Redis/imgs/10-3.jpg" weight="545" height="254">

从服务器psync命令重连之后：
1. 如果offset偏移量之后的数据仍然存在于复制积压缓冲区，那么执行部分重同步操作。
2. 相反，则执行完整重同步操作。

### 运行ID
1. 每个Redis服务器,不论主服务器还是从服务，都会有自己的运行ID
2. 运行ID在服务器启动时自动生成,由40个随机的十六进制字符组成,例如53b9b28df8042fdc9ab5e3fcbbbabff1d5dce2b3

3. 当从服务器对主服务器进行初次复制时,主服务器会将自己的运行ID传送给从服务器,而从服务器则会将这个运行ID保存起来。当从服务器断线并重新连上一个主服务器时,从服务器将向当前连接的主服务器发送之前保存的运行ID:
    - 如果从服务器保存的运行ID和当前连接的主服务器的运行ID相同,那么说明从服务器断线之前复制的就是当前连接的这个主服务器,主服务器可以继续尝试执行部分重同步操作。
    - 相反地,如果从服务器保存的运行ID和当前连接的主服务器的运行ID并不相同,那么说明从服务器断线之前复制的主服务器并不是当前连接的这个主服务器,主服务器将对从服务器执行完整重同步操作。

## 心跳检测
在命令传播阶段,从服务器默认会以每秒一次的频率,向主服务器发送命令:
REPLCONF ACK <replication_offset> <br>
其中replication_offset是从服务器当前的复制偏移量。<br>
发送REPLCONFACK命令对于主从服务器有三个作用：
1. 检测主从服务器的网络连接状态。
2. 辅助实现nin-slaves选项。
3. 检测命令丢失。


## 详细实现步骤
1. 设置主服务器的地址和端口（redisServer有*masterhost和masterport字段用于存储该信息）
2. 建立套接字连接
3. 发送ping命令（检查套接字读写状态是否正常、检查网络等作用）
4. 身份验证
5. 发送端口信息
6. 同步
7. 命令传播

