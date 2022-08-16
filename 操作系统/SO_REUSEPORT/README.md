# SO_REUSEPORT
socket reuse port的意思，启动该选项后，同一用户可绑定相同的端口进行监听，内核对这些监听相同三元组的socket套接字实行负载均衡，将TCP连接请求均匀地分配给这些socket套接字。
- 允许多个套接字 bind()/listen() 同一个TCP/UDP端口
- 每一个线程拥有自己的服务器套接字
- 在服务器套接字上没有了锁的竞争
- 内核层面实现负载均衡
- 安全层面，监听同一个端口的套接字只能位于同一个用户下面

# SO_REUSEADDR
从字面意思理解，SO_REUSEPORT是端口重用，SO_REUSEADDR是地址重用。两者的区别：

（1）SO_REUSEPORT是允许多个socket绑定到同一个ip+port上。SO_REUSEADDR用于对TCP套接字处于**TIME_WAIT**状态下的socket，才可以重复绑定使用。

（2）两者使用场景完全不同。SO_REUSEADDR这个套接字选项通知内核，如果端口忙，但TCP状态位于TIME_WAIT，可以重用端口。这个一般用于当你的程序停止后想立即重启的时候，如果没有设定这个选项，会报错EADDRINUSE，需要等到TIME_WAIT结束才能重新绑定到同一个ip+port上。而SO_REUSEPORT用于多核环境下，允许多个线程或者进程绑定和监听同一个ip+port，无论UDP、TCP（以及TCP是什么状态）。

# 参考资料
[SO_REUSEPORT和SO_REUSEADDR特性](https://zhuanlan.zhihu.com/p/501695927)

[SO_REUSEPORT 负载均衡](https://www.cnblogs.com/dream397/p/14685965.html)