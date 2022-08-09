# MSS
TCP数据包每次能够传输的最大数据分段（不包含头部，如果没有额外选项，头部一般为20字节）。

在TCP三次握手时会协定MSS大小。一般取值关系为：**MSS = MTU - IP header头大小 - TCP 头大小**

每次执行TCP的发送消息函数，会重新计算一次MSS。

# MTU
MTU（Maximum Transmission Unit最大传输单元）是在IP层下面的MAC协议中的概念，MAC协议我们可以理解为是物理层的一些协议，它位于IP协议的下层。

可以认为是IP层每次能够传输的最大数据帧大小，超过这个数值则需要进行分片。一般为1500字节。

# 为什么IP层分片，TCP层还要做分片
数据在TCP分段，就是为了在IP层不需要分片，同时发生重传的时候只重传分段后的小份数据。

# 参考资料
[动图图解！既然IP层会分片，为什么TCP层也还要分段？](https://www.ahfesco.com.cn/affairs/Article.asp?id=3518)

[IP协议详解](https://blog.csdn.net/weixin_45523353/article/details/124477900)