# Goroutine
在Go语言中，每一个并发的执行单元叫作一个goroutine。<br>
当一个程序启动时，其主函数即在一个单独的goroutine中运行，我们叫它main goroutine。新的goroutine会用go语句来创建。在语法上，go语句是一个普通的函数或方法调用前加上关键字go。go语句会使其语句中的函数在一个新创建的goroutine中运行。而go语句本身会迅速地完成。(主函数返回时，所有的goroutine都会被直接打断，即父goroutine结束时，其子goroutine会一并结束)
```
f()    // call f(); wait for it to return
go f() // create a new goroutine that calls f(); don't wait
```
一个goroutine必定对应一个函数，可以创建多个goroutine去执行相同的函数。

# 协程的生命周期
- _Gidle为协程刚开始创建时的状态，当新创建的协程初始化后，会变为_Gdead状态，
- _Gdead状态也是协程被销毁时的状态。
- _Grunnable表示当前协程在运行队列中，正在等待运行。
- _Grunning代表当前协程正在被运行，已经被分配给了逻辑处理器和线程。
- _Gwaiting表示当前协程在运行时被锁定，不能执行用户代码。在垃圾回收及channel通信时经常会遇到这种情况。
- _Gsyscall代表当前协程正在执行系统调用。
- _Gpreempted是Go 1.14新加的状态，代表协程G被强制抢占后的状态。
- _Gcopystack代表在进行协程栈扫描时发现需要扩容或缩小协程栈空间，将协程中的栈转移到新栈时的状态。

<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Golang/imgs/7.jpg" width="568">

# GM模型
早期调度器只有一个全局队列，多个m从同一个全局队列竞争g，造成激烈的**锁竞争**。新创建的g被放入全局队列导致被其他m执行，破坏了局部性。

# GMP模型
在 Go 语言中，每一个 goroutine 是一个独立的执行单元，相较于每个 OS 线程固定分配 2M 内存的模式，goroutine 的栈采取了动态扩容方式， 初始时仅为2KB，随着任务执行按需增长，最大可达 1GB（64 位机器最大是 1G，32 位机器最大是 256M），且完全由 golang 自己的调度器 Go Scheduler 来调度。此外，GC 还会周期性地将不再使用的内存回收，收缩栈空间。 因此，Go 程序可以同时并发成千上万个 goroutine 是得益于它强劲的调度器和高效的内存模型。

<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Golang/imgs/1.png" weight="568" height="449">

- G: 表示 Goroutine，每个 Goroutine 对应一个 G 结构体，G 存储 Goroutine 的运行堆栈、状态以及任务函数，可重用。G 并非执行体，每个 G 需要绑定到 P 才能被调度执行。每个结构体G中有一个sched的属性就是用来保存它上下文的。这样，goroutine 就可以很轻易的来回切换。
- P: Processor，表示逻辑处理器， 对 G 来说，P 相当于 CPU 核，G 只有绑定到 P(在 P 的 local runq 中)才能被调度。对 M 来说，P 提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等，P 的数量决定了系统内最大可并行的 G 的数量（前提：物理 CPU 核数 >= P 的数量），P 的数量由用户设置的 GOMAXPROCS 决定，但是不论 GOMAXPROCS 设置为多大，P 的数量最大为 256。
- M: Machine，OS 线程抽象，代表着真正执行计算的资源，在绑定有效的 P 后，进入 schedule 循环；而 schedule 循环的机制大致是从 Global 队列、P 的 Local 队列以及 wait 队列中获取 G，切换到 G 的执行栈上并执行 G 的函数，调用 goexit 做清理工作并回到 M，如此反复。M 并不保留 G 状态，这是 G 可以跨 M 调度的基础，M 的数量是不定的，由 Go Runtime 调整，为了防止创建过多 OS 线程导致系统调度不过来，目前默认最大限制为 10000 个。m结构体对象主要记录着工作线程的诸如栈的起止位置、当前正在执行的goroutine以及是否空闲等等状态信息之外，还通过指针维持着与p结构体的实例对象之间的绑定关系。

# 调度循环
调度循环指从调度协程g0开始，找到接下来将要运行的协程g、再从协程g切换到协程g0开始新一轮调度的过程。它和上下文切换类似，但是上下文切换关注的是具体切换的状态，而调度循环关注的是调度的流程。

从协程g0调度到协程g，经历了从schedule函数到execute函数再到gogo函数的过程。其中:
- schedule函数处理具体的调度策略，选择下一个要执行的协程；
- execute函数执行一些具体的状态转移、协程g与结构体m之间的绑定等操作；
- gogo函数是与操作系统有关的函数，用于完成栈的切换及CPU寄存器的恢复。

执行完毕后，切换到协程g执行。当协程g主动让渡、被抢占或退出后，又会切换到协程g0进入第二轮调度。在从协程g切换回协程g0时，mcall函数用于保存当前协程的执行现场，并切换到协程g0继续执行，mcall函数仍然是和平台有关的汇编指令。切换到协程g0后会根据切换原因的不同执行不同的函数，例如，如果是用户调用Gosched函数则主动让渡执行权，执行gosched_m函数，如果协程已经退出，则执行goexit函数，将协程g放入p的freeg队列，方便下次重用。执行完毕后，再次调用schedule函数开始新一轮的调度循环，从而形成一个完整的闭环，循环往复。

<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Golang/imgs/5.jpg" width="568">

# 调度策略
调度的核心策略位于schedule函数中。

### 1、runnext获取下一个执行的G。

### 2、P的本地队列获取G。

### 3、全局schedt.runq获取G。
> 当P每执行61次调度，或者局部运行队列中不存在可用的协程时，都需要从全局运行队列中查找一批协程分配给本地运行队列。

### 4、寻找是否有可运行的网络协程。
> runtime.netpoll函数获取当前可运行的协程列表，返回第一个可运行的协程。并通过injectglist函数将其余协程放入全局运行队列等待被调度。

### 5、窃取其他P中的G。
> 将要窃取的P本地运行队列中Goroutine个数的一半放入自己的运行队列中。

### 6、都没有找到可执行的G，则当前的P会解除与M的绑定，P会被放入空闲P队列中，而与P绑定的M没有任务可做，进入休眠状态。

<img src="https://raw.githubusercontent.com/dark-tone/notes/main/Golang/imgs/6.jpg" width="568">


# 阻塞
## 用户态阻塞/唤醒
当 goroutine 因为 channel 操作阻塞时，对应的 G 会被放置到某个 wait 队列(如 channel 的 waitq)，该 G 的状态由_Gruning 变为 _Gwaitting ，而 M 会跳过该 G 尝试获取并执行下一个 G，如果此时没有 runnable 的 G 供 M 运行，那么 M 将解绑 P，并进入 sleep 状态；当阻塞的 G 被另一端的 G2 唤醒时（比如 channel 的可读/写通知），G 被标记为 runnable，尝试加入 G2 所在 P 的 runnext，然后再是 P 的 Local 队列和 Global 队列。

## 系统调用阻塞
当 G 被阻塞在某个系统调用上时，此时 G 会阻塞在 _Gsyscall 状态，M 也处于 block on syscall 状态，此时的 M 可被抢占调度：执行该 G 的 M 会与 P 解绑，而 P 则尝试与其它空闲的 M 绑定，继续执行其它 G。如果没有其它空闲的 M，但 P 的 Local 队列中仍然有 G 需要执行，则创建一个新的 M；当系统调用完成后，G 会重新尝试获取一个空闲的 P 进入它的 Local 队列恢复执行，如果没有空闲的 P，G 会被标记为 runnable 加入到 Global 队列。


# 参考资料
《Go语言底层原理剖析》

[Go学习总结之GPM模型](https://zhuanlan.zhihu.com/p/261807834)

[GoLang GPM模型](https://studygolang.com/articles/29227?fr=sidebar)

[GPM](https://blog.csdn.net/dream_1996/article/details/118051217)

[也谈goroutine调度器](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)

[go GMP调度原理](https://blog.csdn.net/weixin_41565755/article/details/122443114)

[深入 golang -- GMP调度](https://zhuanlan.zhihu.com/p/502740833)

[Go GPM 调度器介绍](https://blog.csdn.net/Rojer_Gz/article/details/126389051)