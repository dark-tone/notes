# 输入网址到显示页面发生了什么
1. 你向浏览器的地址栏输入一个域名，如http://www.baidu.com
2. 发起域名解析过程，获取真实IP地址。
3. 拿到ip地址之后，发起TCP 握手(3次)。
4. 握手成功，构造request,即 HTTP 中request请求.并发送到目的地。
5. 服务器接受到一个完整的request(该边界的指定一般是conten-length,chunked也有),根据用户的request内容运算出相应的response。
6. 服务器将response 沿着request建立的连接，向浏览器(客户端)发送数据。
7. keepalive的时候不关闭该连接，没有keepalive的时候发起tcp close,4次握手
8. 浏览器根据接收到的response开始渲染页面。

至此，一个网页的打开过程完毕，我们从中提取出耗时的部分。
1. DNS查询时间(一来一回,走UDP协议) 网络IO
2. tcp 建立连接握手 网络IO
3. request构造时间(cpu运算)
4. request发送完毕时间(网络IO)
5. 服务器接收request运算构造response(CPU运算,特指构造response过程中没有任何IO操作)
6. 服务器发送response到客户端的时间(网络IO)
7. 服务器关闭连接时间(IO)
8. 客户端接收数据渲染页面时间(cpu运算)。