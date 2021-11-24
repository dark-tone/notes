# 概述
HTTPS （全称：Hyper Text Transfer Protocol over SecureSocket Layer），是以安全为目标的 HTTP 通道，在HTTP的基础上通过传输加密和身份认证保证了传输过程的安全性。

# SSL/TLS
**SSL**：（Secure Socket Layer，安全套接字层），位于可靠的面向连接的网络层协议和应用层协议之间的一种协议层。SSL通过互相认证、使用数字签名确保完整性、使用加密确保私密性，以实现客户端和服务器之间的安全通讯。该协议由两层组成：SSL记录协议和SSL握手协议。

**TLS**：（Transport Layer Security，传输层安全协议），用于两个应用程序之间提供保密性和数据完整性。该协议由两层组成：TLS记录协议和TLS握手协议。

TLS/SSL均为一种加密通道的规范，SSL基本已弃用，目前主流使用TLS2.0。

# 过程
<img src="https://raw.githubusercontent.com/dark-tone/notes/main/%E7%BD%91%E7%BB%9C%E7%9B%B8%E5%85%B3/imgs/5.png" weight="567" height="362">

1. 客户端向服务端发出加密通信的请求。
>- 支持的协议版本，比如TLS 1.0版。
>- 一个客户端生成的随机数 client random，稍后用于生成"对话密钥"。
>- 支持的加密方法，比如RSA公钥加密。
>- 支持的压缩方法。

2. 服务端收到请求并响应。
>- 确认使用的加密通信协议版本，比如TLS 1.0版本。如果浏览器与服务器支持的版本不一致，服务器关闭加密通信。
>- 一个服务器生成的随机数server random，稍后用于生成"对话密钥"。
>- 确认使用的加密方法，比如RSA公钥加密。
>- 服务器证书。

3. 客户端验证证书有效性。
>- **日期检测** 首先，浏览器检查证书的起始日期和结束日期，以确保证书仍然有效。如果证书 过期了，或者还未被激活，则证书有效性验证失败，浏览器显示一条错误信息。
>- **签名颁发者可信度检测** 每个证书都是由某些 证书颁发机构(CA) 签发的，它们负责为服务器担保。证书有不同的等级，每种证书都要求不同级别的背景验证。比如，如果申请某个电 子商务服务器证书，通常需要提供一个营业的合法证明。
任何人都可以生成证书，但有些 CA 是非常著名的组织，它们通过非常清晰的流 程来验证证书申请人的身份及商业行为的合法性。因此，浏览器会附带一个签名颁发机构的受信列表。如果浏览器收到了某未知(可能是恶意的)颁发机构签发的证书，那它通常会显示一条警告信息。
>- **签名检测** 一旦判定签名授权是可信的，浏览器就要对签名使用签名颁发机构的公开密钥， 并将其与校验码进行比较，以查看证书的完整性。
>- **站点身份检测** 为防止服务器复制其他人的证书，或拦截其他人的流量，大部分浏览器都会试着 去验证证书中的域名与它们所对话的服务器的域名是否匹配。服务器证书中通常 都包含一个域名，但有些 CA 会为一组或一群服务器创建一些包含了服务器名称 列表或通配域名的证书。如果主机名与证书中的标识符不匹配，面向用户的客户 端要么就去通知用户，要么就以表示证书不正确的差错报文来终止连接。

4. 客户端生成随机数(PreMaster secret)，并将随机数用公钥加密传输给服务端。
>PreMaster Secret是在客户端使用RSA或者Diffie-Hellman等加密算法生成的。它将用来跟服务端和客户端在Hello阶段产生的随机数结合在一起生成 Master Secret。在客户端使用服务端的公钥对PreMaster Secret进行加密之后传送给服务端，服务端将使用私钥进行解密得到PreMaster secret。也就是说服务端和客户端都有一份相同的PreMaster secret和随机数。
PreMaster secret前两个字节是TLS的版本号，这是一个比较重要的用来核对握手数据的版本号，因为在Client Hello阶段，客户端会发送一份加密套件列表和当前支持的SSL/TLS的版本号给服务端，而且是使用明文传送的，如果握手的数据包被破解之后，攻击者很有可能串改数据包，选择一个安全性较低的加密套件和版本给服务端，从而对数据进行破解。所以，服务端需要对密文中解密出来对的PreMaster版本号跟之前Client Hello阶段的版本号进行对比，如果版本号变低，则说明被串改，则立即停止发送任何消息。

5. 客户端和服务端根据约定的加密方法，使用前面的三个随机数，生成对话密钥（session key），用来加密接下来的整个对话过程。


# 证书链
事实上，证书的验证过程中还存在一个证书信任链的问题，因为我们向 CA 申请的证书一般不是根证书签发的，而是由中间证书签发的，比如百度的证书，对于这种三级层级关系的证书的验证过程如下：客户端收到 http://baidu.com 的证书后，发现这个证书的签发者不是根证书，就无法根据本地已有的根证书中的公钥去验证 http://baidu.com 证书是否可信。于是，客户端根据 http://baidu.com 证书中的签发者，找到该证书的颁发机构是 “GlobalSign Organization Validation CA - SHA256 - G2”，然后向 CA 请求该中间证书。请求到证书后发现 “GlobalSign Organization Validation CA - SHA256 - G2” 证书是由 “GlobalSign Root CA” 签发的，由于 “GlobalSign Root CA” 没有再上级签发机构，说明它是根证书，也就是自签证书。应用软件会检查此证书有否已预载于根证书清单上，如果有，则可以利用根证书中的公钥去验证 “GlobalSign Organization Validation CA - SHA256 - G2” 证书，如果发现验证通过，就认为该中间证书是可信的。“GlobalSign Organization Validation CA - SHA256 - G2” 证书被信任后，可以使用 “GlobalSign Organization Validation CA - SHA256 - G2” 证书中的公钥去验证 http://baidu.com 证书的可信性，如果验证通过，就可以信任 http://baidu.com 证书。在这四个步骤中，最开始客户端只信任根证书 GlobalSign Root CA 证书的，然后 “GlobalSign Root CA” 证书信任 “GlobalSign Organization Validation CA - SHA256 - G2” 证书，而 “GlobalSign Organization Validation CA - SHA256 - G2” 证书又信任 http://baidu.com 证书，于是客户端也信任 http://baidu.com 证书。

# 为什么要结合使用对称加密和非对称加密？
- 一般HTTP客户端与服务端存在大量通信，但非对称加密的加解密效率比对称加密低。
- 单独使用对称加密，无法实现密钥交换。
- 单独使用非对称加密，加密通道只能是单向的，如果黑客拥有public key，则可以解密服务端发送到客户端的信息。

# 每次连接都需要进行HTTPS握手吗？
浏览器客户端访问同一个https服务器，可以不必每次都进行完整的TLS Handshake，因为完整的TLS Handshake，涉及到认证服务器的身份（数字证书），需要做大量的非对称加密/解密运算，此外还需要做伪随机函数PRF，通过“Pre-Master Key”、“Server Nonce”、“Client Nonce”共同推导出session key，非对称加密算法RSA/DSA非常耗费CPU资源。为了克服这个困难，服务器维护一个以session ID为索引的结构体，用于临时存放session key，并在TLS handshake 阶段分享给浏览器。当浏览器重新连接https 服务器时，TLS handshake 阶段，出示自己的session ID，服务器获得session ID，以此为索引，可以获得和该浏览器共同拥有的session key，使用session key可以直接对用户流量做加密/解密动作。

# 参考文章
[HTTPS、SSL、TLS三者之间的联系和区别](https://blog.csdn.net/enweitech/article/details/81781405)

[HTTPS系列干货（一）：HTTPS 原理详解](https://zhuanlan.zhihu.com/p/27395037)

[Https流程和原理](https://www.jianshu.com/p/b0b6b88fe9fe)

[HTTPS必须在每次请求中都要先在SSL层进行握手传递秘钥吗？](https://www.zhihu.com/question/67740663/answer/256288406)

[HTTPS详解](https://segmentfault.com/a/1190000011675421)

[浏览器如何验证HTTPS证书的合法性？](https://www.zhihu.com/question/37370216)