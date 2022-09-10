# 创建
使用<code>make()</code>方法可创建<code>channel</code>，make方法在编译时会替换为<code>makechan</code>方法，<code>makechan</code>方法会返回<code>*hchan</code>类型。该类型即<code>chan</code>类型的底层实现。
> 可编写创建channel的go方法，然后使用<code>go tool compile -S xxx.go > xxx.c </code>生成对应的汇编代码，可看见调用的对应的makechan方法。

创建时会做一些检查:
- 元素大小不能超过 64K
- 元素的对齐大小不能超过 maxAlign 也就是 8 字节
- 计算出来的内存是否超过限制

创建时的策略:
- 如果是无缓冲的 channel，会直接给 hchan 分配内存
- 如果是有缓冲的 channel，并且元素不包含指针，那么会为 hchan 和底层数组分配一段连续的地址
- 如果是有缓冲的 channel，并且元素包含指针，那么会为 hchan 和底层数组分别分配地址

# 底层结构
``` go
// src/runtime/chan.go

type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```
**buf**：循环数组/环形缓冲区，当创建的是有缓冲的channel时使用。

**sendx、recvx**：用于标明循环数组头尾。

**recvq、sendq**：接收等待队列，发送等待队列。

**lock**：用于处理接收与发送的并发竞争问题。

# 发送
编译时转换为runtime.chansend函数。

- 如果 channel 的读等待队列存在接收者goroutine
    - 将数据直接发送给第一个等待的 goroutine， 唤醒接收的 goroutine

- 如果 channel 的读等待队列不存在接收者goroutine
    - 如果循环数组buf未满，那么将会把数据发送到循环数组buf的队尾
    - 如果循环数组buf已满，这个时候就会走阻塞发送的流程，将当前 goroutine 加入写等待队列，并挂起等待唤醒

# 接收
编译时转换为runtime.chanrecv函数。

- 如果 channel 的写等待队列存在发送者goroutine
    - 如果是无缓冲 channel，直接从第一个发送者goroutine那里把数据拷贝给接收变量，唤醒发送的 goroutine
    - 如果是有缓冲 channel（已满），将循环数组buf的队首元素拷贝给接收变量，将第一个发送者goroutine的数据拷贝到 buf循环数组队尾，唤醒发送的 goroutine

- 如果 channel 的写等待队列不存在发送者goroutine
    - 如果循环数组buf非空，将循环数组buf的队首元素拷贝给接收变量
    - 如果循环数组buf为空，这个时候就会走阻塞接收的流程，将当前 goroutine 加入读等待队列，并挂起等待唤醒

# 关闭
编译时转换为runtime.closechan函数。

# 参考资料
[Go底层原理 - channel的底层实现原理？](https://juejin.cn/post/7129085275266875422)