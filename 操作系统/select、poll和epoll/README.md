# select
``` C
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)
```
select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述符就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以通过**遍历fdset**，来找到就绪的描述符。


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
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

红黑树，链表


```
int s = socket(AF_INET, SOCK_STREAM, 0);    
bind(s, ...) 
listen(s, ...) 
 
int epfd = epoll_create(...); 
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中 
 
while(1){ 
    int n = epoll_wait(...) 
    for(接收到数据的socket){ 
        //处理 
    } 
```


## 工作模式
- LT（level trigger水平模式）<br>
当epoll_wait检测到文件描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
- ET（edge trigger边缘模式）<br>
当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。
> LT和ET各有优缺点。使用哪种模式取决于并发量。当并发量比较小时，比较推荐LT，因为LT模式下应用的读写逻辑比较简单，不容易遗漏事件，代码不易出错好维护，而且性能损失不大。当并发量非常大时，推荐使用ET模式，可以有效提升EPOLL效率。


# 三者区别
- select和poll每次调用函数需要将监控的fds从用户空间拷贝到内核空间，epoll则不用
- select和poll有事件时，还需要遍历文件符

# 参考资料
[Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)
[epoll的LT和ET](https://www.jianshu.com/p/d3442ff24ba6)