# 实现分布式锁

### 基本命令
```
SET key value [EX seconds] [PX milliseconds] [NX|XX]
// 例：SET lock 1 EX 10 NX
```
- EX second ：设置键的过期时间为 second 秒。 SET key value EX second 效果等同于 SETEX key second value 。
- PX millisecond ：设置键的过期时间为 millisecond 毫秒。 SET key value PX millisecond 效果等同于 PSETEX key millisecond value 。
- NX ：只在键不存在时，才对键进行设置操作。 SET key value NX 效果等同于 SETNX key value 。
- XX ：只在键已经存在时，才对键进行设置操作。

设置键与设置过期时间为原子操作。

### 基本过程
1. 客户端a使用SET + NX参数设置锁，设置成功则获得锁（结果返回1）。
2. 客户端b使用SET + NX参数设置锁，设置失败，进入失败的处理逻辑。
3. 客户端a事务完成，使用DEL释放锁。此时其他客户端可获得锁来处理逻辑。

### 如何防止锁过期了，但是事务还没完成？
加入watch dog机制，比如添加一个定时任务，监控当前锁的过期时间，如果即将过期但事务仍未完成，则延长锁的过期时间。

### 如何防止释放了非自己持有的锁？
set的value设置为客户端的唯一标志uuid，释放锁时先判断是否是自己所持有的锁。

### 如何防止不断尝试获得锁的情况出现？
可加入消息订阅机制，客户端a释放锁后，发布一个锁释放的消息，其余订阅了消息的客户端再尝试获得锁。

### 如何实现同一客户端可重入机制？
使用hash表记录客户端 - 锁数量的记录，在设置锁的同时，在hash表对应记录上加1，释放锁时，先判断hash表对应客户端的锁数量是否>0，大于0则锁数量减1，否则直接删除键。

### RedLock问题
