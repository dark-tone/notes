# TCP相关流程图
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/imgs/12.jpg" width="500px">

# 相关函数
## socket
``` C
int socket(int domain, int type, int protocol);
// domain：指定通信域
// type：指定套接字类型
// protocol：附加协议，如果type使用的是 SOCK_STREAM 或者 SOCK_DGRAM此时该参数可以指定为 0

// return
// 成功：套接字描述符
// 失败：-1
```
socket函数对应于普通文件的打开操作。普通文件的打开操作返回一个文件描述字，而socket()用于创建一个socket描述符（socket descriptor），它唯一标识一个socket。这个socket描述字跟文件描述字一样，后续的操作都有用到它，把它作为参数，通过它来进行一些读写操作。


## bind
``` C
int bind(int sockfd, const struct sockaddr* addr, socklen_t addrlen);
/*
sockfd: socket函数返回的文件描述符
addr：网络信息结构体
addrlen：网络信息结构体的大小

return
成功：0
失败：-1
*/
```
bind 函数主要是服务器端使用，把一个本地协议地址赋予套接字；socket 函数并没有为套接字绑定本地地址和端口号，对于服务器端则必须显性绑定地址和端口号。

通常服务器在启动的时候都会绑定一个众所周知的地址（如ip地址+端口号），用于提供服务，客户就可以通过它来接连服务器；而客户端就不用指定，有系统自动分配一个端口号和自身的ip地址组合。这就是为什么通常服务器端在listen之前会调用bind()，而客户端就不会调用，而是在 connect()时由系统随机生成一个。

## listen
``` C
int listen(int sockfd, int backlog);
/*
sockfd: socket函数返回的文件描述符
backlog：允许 同时连接到服务器的客户端的个数(设置成 5 或者 10 都可以 只要不是0就行)。

return
成功：0
失败：-1
*/
```
服务端绑定端口之后调用listen()来监听这个socket。

## connect
``` C
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
/*
sockfd: socket函数返回的文件描述符
addr：服务器的网络信息结构体的地址
addrlen：addr的长度

return
成功：0
失败：-1
*/
```
客户端发出连接请求调用的函数。

## accept
``` C
int accept(int socket, struct sockaddr *restrict address,socklen_t *restrict address_len);
/*
sockfd: socket函数返回的文件描述符
address：用来保存客户端的网络信息结构体的地址，如果不需要，可以传NULL
address_len:客户端的网络信息结构体的长度，如果不需要，可以传NULL

return
成功：返回一个新的文件描述符，专门用于和该客户端通信
失败：-1
*/
```
TCP服务器端依次调用socket()、bind()、listen()之后，就会监听指定的socket地址了。TCP客户端依次调用socket()、 connect()之后就想TCP服务器发送了一个连接请求。TCP服务器监听到这个请求之后，就会调用accept()函数取接收请求，这样连接就建立好了。之后就可以开始网络I/O操作了，即类同于普通文件的读写I/O操作。

## read、write等函数
网络I/O相关函数有几组：
- read()/write()
- recv()/send()
- readv()/writev()
- recvmsg()/sendmsg()
- recvfrom()/sendto()

在 Linux 和 Windows 平台下，使用不同的函数发送和接收socket函数。因为Linux 不区分套接字文件和普通文件，使用 write() 可以向套接字中写入数据，使用 read() 可以从套接字中读取数据。而Windows 区分普通文件和套接字，并定义了专门的接收和发送的函数，即send() 函数和recv()函数。
``` C
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
/*
从 fd 文件中读取 count 个字节并保存到缓冲区 buf

return
成功：返回读取到的字节数（但遇到文件结尾则返回0）
失败：返回 -1。
*/
ssize_t write(int fd, const void *buf, size_t count);
/*
fd 为要写入的文件的描述符
buf 为要写入的数据的缓冲区地址
count 为要写入的数据的字节数。

return
成功：写入的字节数
失败：-1
*/

#include <sys/types.h>
#include <sys/socket.h>

ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);

ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);

ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```


## close
``` C
int close(int fd);
```
在服务器与客户端建立连接之后，会进行一些读写操作，完成了读写操作就要关闭相应的socket描述字，好比操作完打开的文件要调用fclose关闭打开的文件。

注意：close操作只是使相应socket描述字的引用计数-1，只有当引用计数为0的时候，才会触发TCP客户端向服务器发送终止连接请求。


# 参考资料
[Linux socket的基本操作socket、bind、listen、accept](https://zhuanlan.zhihu.com/p/365478112)

[socket编程基本函数](https://blog.csdn.net/cyhhh/article/details/125174265)

[与socket网络编程有关的函数](https://blog.csdn.net/oqqHuTu12345678/article/details/125728148)