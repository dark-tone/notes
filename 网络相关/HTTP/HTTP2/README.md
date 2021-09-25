# HTTP2

## 二进制分帧层(Binary Framing Layer)
HTTP2在应用层(HTTP/2)和传输层(TCP or UDP)之间增加一个二进制分帧层。

在二进制分帧层中， HTTP/2 会将所有传输的信息分割为更小的消息和帧（frame）,并对它们采用二进制格式的编码 ，其中 HTTP1.x 的首部信息会被封装到 HEADER frame，而相应的 Request Body 则封装到 DATA frame 里面。


HTTP/2 通信都在一个连接上完成，这个连接可以承载任意数量的双向数据流。

### 帧格式
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/%E7%BD%91%E7%BB%9C%E7%9B%B8%E5%85%B3/imgs/5.jpg" weight="562" height="175">

- Length：代表整个 frame 的长度，用一个 24 位无符号整数表示
>但是这不意味着就能处理 2^24 16M大小的帧，一般是默认只支持2^16 16k以下的帧，而2^16 - 2^24 16M 的帧 需要接收端公布自己可以处理这么大的帧，需要在 SETTINGS_MAX_FRAME_SIZE 帧中告知。

- Type： 定义 frame 的类型。帧类型决定了帧主体的格式和语义，如果 type 为 unknown 应该忽略或抛弃。

|帧类型 |	编码类型 |	用途|
| :---: | :---: | --- |
|DATA| 	0x0| 	传递HTTP包体|
|HEADERS |	0x1 |	传递HTTP包头|
|PRIORITY 	|0x2| 	指定Stream 流的优先级|
|RST_STREAM 	|0x3| 	终止Stream流|
|SETTINGS 	|0x4| 	修改连接或者Stream流的配置|
|PUSH_PROMISE 	|0x5| 	服务端推送资源时描述请求的帧|
|PING 	|0x6| 	心跳监测兼具测量RTT的功能|
|GOAWAY 	|0x7| 	优雅的终止错误或通知错误|
|WINDOW_UPDATE 	|0x8| 	实现流量控制|
|CONTINUATION 	|0x9| 	传递较大HTTP头部时的持续帧|

- Flags :是为帧类型相关而预留的布尔标识。标识对于不同的帧类型赋予了不同的语义。

- R: 是一个保留的比特位。这个比特的语义没有定义，发送时它必须被设置为 (0x0), 接收时需要忽略。

- Frame Payload : 是主体内容，由帧类型决定。

## 头部压缩(HPACK算法)
1. HTTP/2在客户端和服务器端使用“首部表”来跟踪和存储之前发送的键－值对，对于相同的数据，不再通过每次请求和响应发送（仅发送索引）。
2. 首部表在HTTP/2的连接存续期内始终存在，由客户端和服务器共同渐进地更新。
3. 每个新的首部键－值对要么被追加到当前表的末尾，要么替换表中之前的值。

### 静态表
静态表很简单，只包含已知的header字段。分为两种：
1. name和value都可以完全确定，比如:metho: GET、:status: 200
2. 只能够确定name：比如:authority、cookie
>第一种情况很好理解，已知键值对直接使用一个字符表示；<br>
第二种情况稍微说明下：首先将name部分先用一个字符（比如cookie）来表示，同时，根据情况判断是否告知服务端，将 cookie: xxxxxxx 添加到动态表中（我们这里默认假定是从客户端向服务端发送消息）

### 动态表
- 动态表最初是一个空表，当每次解压头部的时候，有可能会添加条目（比如前面提到的cookie，当解压过一次cookie时，cookie: xxxxxxx就有可能被添加到动态表了，至于是否添加要根据后面提到的指令判断）。
- 动态表允许包含重复的条目，也就是可能出现完全相同的键值对。
- 为了限制解码器的需求，动态表大小有严格限制的。


## 多路复用(MultiPlexing)
在一个 TCP 连接上，我们可以向对方不断发送帧，每帧的 stream identifier 的标明这一帧属于哪个流，然后在对方接收时，根据 stream identifier 拼接每个流的所有帧组成一整块数据。

把 HTTP/1.1 每个请求都当作一个流，那么多个请求变成多个流，请求响应数据分成多个帧，不同流中的帧交错地发送给对方，这就是 HTTP/2 中的多路复用。

流的概念实现了单连接上多请求-响应并行，解决了**队头阻塞**的问题，减少了TCP 连接数量和TCP连接慢启动（因为TCP连接共用了，无需重复创建和释放）造成的问题，所以 http2 对于同一域名只需要创建一个连接，而不是像 http/1.1 那样创建 6~8 个连接。
> ### 流(stream)
>因为一个TCP连接可以传输多个 请求/响应，每个 请求/响应 包含多个帧，因此需要区分哪些帧属于哪个 请求/响应，同一个请求的帧组合起来就是一个“流”，每个流都有自身的ID，客户端使用奇数的ID，服务器使用偶数的ID，ID不可以重复使用，每次发起新的 请求的时候，都会使用一个更大的ID。
> ### 队头阻塞
> 当顺序发送的请求序列中的一个请求因为某种原因被阻塞时，在后面排队的所有请求也一并被阻塞，会导致客户端迟迟收不到数据。
> ### TCP慢启动
> TCP连接在一开始不能发送大尺寸的数据包，如果数据传输成功，后续才会逐步提高传输的速度（TCP拥塞控制）。


## 服务端推送(Server Push)
在浏览器刚请求HTML的时候就提前把可能会用到的JS、CSS文件发给客户端，减少等待的延迟，这被称为"服务器推送"，如果服务端推送的资源已经被浏览器缓存过，浏览器可以通过发送RST_STREAM帧来拒收。主动推送也遵守同源策略，换句话说，服务器不能随便将第三方资源推送给客户端，而必须是经过双方确认才行。

## HTTP/2协议协商过程

1. 客户端发起请求，只有请求报头：
```
    GET / HTTP/1. 1
    Host: server. example. com
    Connection: Upgrade, HTTP2-Settings
    Upgrade: h2c
    HTTP2-Settings: <base64url encoding of HTTP/2 SETTINGS payload>
```
2. 服务器若不支持HTTP/2，直接按照HTTP/1.1响应即可
```
    HTTP/1. 1 200 OK
    Content-Length: 243
    Content-Type: text/html
    . . .
```
3. 服务器支持HTTP/2，通知客户端一起切换到HTTP/2协议下
```
    HTTP/1. 1 101 Switching Protocols
    Connection: Upgrade
    Upgrade: h2c

    [ HTTP/2 connection . . .
```
4. 101响应空行之后，服务器必须发送的第一个帧为SETTINGS帧（其负载可能为空）作为连接序言
5. 客户端接收到101响应后，也必须发送一个序言作为响应，其逻辑结构：
```
    PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n // 纯字符串表示，翻译成字节数为24个字节
    SETTINGS帧                       // 其负载可能为空
```
6. 客户端可以马上发送请求帧或其它帧过去，不用等待来自服务器端的SETTINGS帧
7. 任一端接收到SETTINGS帧之后，都需要返回一个包含确认标志位SETTIGN作为确认
8. 其它帧正常传输


# 参考文章
[HTTP2 详解](https://www.jianshu.com/p/e57ca4fec26f)

[透视HTTP协议](https://time.geekbang.org/column/article/112036)

[HTTP/2 相比 1.0 有哪些重大改进？](https://www.zhihu.com/question/34074946)

[HTTP2协议详解](https://www.codercto.com/a/34433.html)

[HTTP/2笔记之连接建立](http://www.blogjava.net/yongboy/archive/2015/03/18/423570.html)

[深入HTTP/2（帧格式）](https://www.jianshu.com/p/e22fef60a7f0)

[详解http-2头部压缩算法](https://segmentfault.com/a/1190000017011816)