# 内存模型
为了保证共享内存的正确性（可见性、有序性、原子性），内存模型定义了共享内存系统中多线程程序读写操作行为的规范。通过这些规则来规范对内存的读写操作，从而保证指令执行的正确性。

## Happens Before
在一个goroutine中，读和写必须按照程序指定的顺序执行。也就是说，只有当内存重排没有改变既定的代码的逻辑顺序时，编译器和处理器才可以重新排序在单个goroutine中执行的读写操作。

如果A happens before B，那么A的执行结果对B可见（并不一定表示A比B先执行，如果A与B执行的顺序对结果没有影响是可以重排序的，说白了，就是能够保证不管CPU，编译器怎么优化，代码从结果看按顺序执行的。）

## Happens Before规则
### Init函数
- 如果包P1中导入了包P2，则P2中的init函数Happens Before 所有P1中的操作
- main函数Happens After 所有的init函数

### Goroutine
- Goroutine的创建Happens Before所有此Goroutine中的操作
- Goroutine的销毁Happens After所有此Goroutine中的操作

### Channel
- 对一个元素的send操作Happens Before对应的receive完成操作
- 对channel的close操作Happens Before receive 端的收到关闭通知操作
- 对于Unbuffered Channel，对一个元素的receive 操作Happens Before对应的send完成操作
- 对于Buffered Channel，假设Channel 的buffer 大小为C，那么对第k个元素的receive操作，Happens Before第k+C个send完成操作。

### Lock
Go里面有Mutex和RWMutex两种锁，RWMutex除了支持互斥的Lock/Unlock，还支持共享的RLock/RUnlock。
- 对于一个Mutex/RWMutex，设n < m，则第n个Unlock操作Happens Before第m个Lock操作。
- 对于一个RWMutex，存在数值n，RLock操作Happens After 第n个UnLock，其对应的RUnLockHappens Before 第n+1个Lock操作

### Once
once.Do中执行的操作，Happens Before 任何一个once.Do调用的返回。

# 参考文章
[Golang内存模型](https://studygolang.com/articles/22523)

[Golang内存模型(Memory Model) ](https://www.cnblogs.com/tttlv/p/14727697.html)

[Go的内存模型](https://www.jianshu.com/p/5e44168f47a3)