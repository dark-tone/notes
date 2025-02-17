# select
``` C
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)
```
select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述符就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以通过**遍历fdset**，来找到就绪的描述符。

return：表示此时有多少个监控的描述符就绪，若超时则为0，出错为-1。


# poll
``` C
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```
不同于select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。
``` C
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```
pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要**轮询pollfd**来获取就绪的描述符。

# epoll
``` C
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；// 负责管理这个池子里的 fd 增、删、改
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

## 示意图
使用红黑树以及双端链表实现。

<img src="https://raw.githubusercontent.com/dark-tone/notes/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/imgs/640.png" style="width:70%">

```
int s = socket(AF_INET, SOCK_STREAM, 0);    
bind(s, ...) 
listen(s, ...) 
 
int epfd = epoll_create(...); 
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中 
 
while(1) {
    int n = epoll_wait(...) 
    for(接收到数据的socket){ 
        //处理 
    } 
```

pollevent结构
```
struct eventpoll {
　　...
   // 红黑树的根节点指针, 存储着所有的事件, 你也可以理解为所有的socket连接都在红黑树中存储
　　struct rb_root rbr;
　　// 双向链表, 存储所有就绪的socket的连接, epoll_wait的返回值就是从这个地方读取的
　　struct list_head rdllist;
　　...
};
```

## 底层原理
使用文件系统的 poll 机制让上层能直接告诉底层，我这个 fd 一旦读写就绪了，请底层硬件（比如网卡）回调的时候自动把这个 fd 相关的结构体放到指定队列中，并且唤醒操作系统。
> 举个例子：网卡收发包其实走的异步流程，操作系统把数据丢到一个指定地点，网卡不断的从这个指定地点掏数据处理。请求响应通过中断回调来处理，中断一般拆分成两部分：硬中断和软中断。poll 函数就是把这个软中断回来的路上再加点料，只要读写事件触发的时候，就会立马通知到上层，采用这种事件通知的形式就能把浪费的时间窗就完全消失了。

poll调用实现了什么？
1. 把事件就绪的 fd 对应的结构体放到一个特定的队列（就绪队列，ready list）
2. 唤醒 epoll


## 工作模式
- LT（level trigger水平模式）<br>
当epoll_wait检测到文件描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
- ET（edge trigger边缘模式）<br>
当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。
> LT和ET各有优缺点。使用哪种模式取决于并发量。当并发量比较小时，比较推荐LT，因为LT模式下应用的读写逻辑比较简单，不容易遗漏事件，代码不易出错好维护，而且性能损失不大。当并发量非常大时，推荐使用ET模式，可以有效提升EPOLL效率。


# 三者区别
- select和poll每次调用函数需要将监控的fds从用户空间拷贝到内核空间，epoll则不用
- select和poll有事件时，还需要遍历文件符

# 文件描述符fd
文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。

文件描述符在形式上是一个**非负整数**。实际上，它是一个索引值，指向**内核为每一个进程所维护的该进程打开文件的记录表**。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。

# 参考资料
[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)

[epoll的LT和ET](https://www.jianshu.com/p/d3442ff24ba6)

[深入理解 Linux 的 epoll 机制](https://mp.weixin.qq.com/s?__biz=MzU0OTE4MzYzMw==&mid=2247515011&idx=2&sn=3812f80dd80bb27340d5849df8d1cec0&chksm=fbb1327dccc6bb6bfd5ab7f9da23220ade44e88e2f8d2506b7e0868bb84665a95f026eddb82d&scene=27)

[揭开 epoll 面纱：Nginx，Redis 等都在用的多路复用，到底是什么？](https://xie.infoq.cn/article/6e0e4822754f224cd11f8ed4e)

[[内核源码] epoll lt / et 模式区别](https://zhuanlan.zhihu.com/p/441677252)
