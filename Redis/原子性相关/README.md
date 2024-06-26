# 场景
执行的多条命令需要确保同时执行。

# redis事务特性
- 如果在执行 exec 之前事务中断了，那么所有的命令都不会执行
- 如果某个命令语法错误，不仅会导致该命令入队失败，整个事务都将无法执行
- 如果执行了 exec 命令之后，那么所有的命令都会按序执行
- 当 redis 在执行命令时，如果出现了错误，那么 redis 不会终止其它命令的执行，这是与关系型数据库事务最大的区别，redis 事务不会因为某个命令执行失败而回滚

# 前提
redis执行是**单线程**的，这意味着如果命令可以确保顺序执行，则不存在并发的影响。

# 实现方法
## 事务
事务是实现在服务器端的行为，用户执行MULTI命令时，服务器会将对应这个用户的客户端对象设置为一个特殊的状态，在这个状态下后续用户执行的查询命令不会被真的执行，而是被服务器缓存起来，直到用户执行EXEC命令为止，服务器会将这个用户对应的客户端对象中缓存的命令按照提交的顺序依次执行。

### 事务缺陷：
#### 不满足原子性
与关系型数据库的事务不同，redis 事务是不满足原子性的，一个事务执行过程中，其他事务或 client 是可以对相应的 key 进行修改的
> 在事务执行 EXEC 命令之前 ，Redis key 依然可以被修改。

想要避免这样的并发性问题就需要使用 WATCH 命令，但是通常来说，必须经过仔细考虑才能决定究竟需要对哪些 key 进行 WATCH 加锁

额外的 WATCH 会增加事务失败的可能，而缺少必要的 WATCH 又会让我们的程序产生竞争条件

#### 后执行的命令无法依赖先执行命令的结果
由于事务中的所有命令都是互相独立的，在遇到 exec 命令之前并没有真正的执行，所以我们无法在事务中的命令中使用前面命令的查询结果

我们唯一可以做的就是通过 watch 保证在我们进行修改时，如果其它事务刚好进行了修改，则我们的修改停止，然后应用层做相应的处理

#### 事务中的每条命令都会与 redis 服务器进行网络交互
redis 事务开启之后，每执行一个操作返回的都是 queued，这里就涉及到客户端与服务器端的多次交互

明明是需要一次批量执行的 n 条命令，还需要通过多次网络交互，显然非常浪费

## pipeline
客户端行为，对redis服务端透明，客户端将命令打包一起发送给服务端执行。

当通过pipeline提交的查询命令数据较少，可以被内核缓冲区所容纳时，Redis可以保证这些命令执行的原子性。然而一旦数据量过大，超过了内核缓冲区的接收大小，那么命令的执行将会被打断，原子性也就无法得到保证。

## lua脚本
好处：
- 减少网络开销。将多个请求通过脚本的形式一次发送，减少网络时延。
- 原子操作。Redis会将整个脚本作为一个整体执行，中间不会被其他命令插入。
- 复用。客户端发送的脚本会永久存在 Redis 中，其他客户端可以复用这一脚本而不需要使用代码完成相同的逻辑。
- 支持后面的步骤依赖前面步骤的结果。

# 参考资料
[Redis事务与lua](https://whiteccinn.github.io/2020/06/02/Redis/redis%E4%BA%8B%E5%8A%A1%E5%92%8Clua/)

[Redis中的pipeline和事务](https://www.zhihu.com/question/422433905/answer/1677796925)

[一文讲透 Redis 事务](https://juejin.cn/post/7219656530236145724)