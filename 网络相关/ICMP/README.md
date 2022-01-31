# 概念
ICMP 全称 Internet Control Message Protocol，就是互联网控制报文协议。它是**TCP/IP协议簇的一个子协议**，用于在IP主机、路由器之间传递控制消息。控制消息是指网络通不通、主机是否可达、路由是否可用等网络本身的消息。工作于网络层。

# 报文格式
8位类型+8位代码+16位检验和+特有内容

# 报文类型
|TYPE	|CODE	|Description	|Query	|Error|
|----|----|----|----|----|
|0	|0	|Echo Reply——回显应答（Ping应答）	|x	| |
|3	|0	|Network Unreachable——网络不可达	| 	|x|
|3	|1	|Host Unreachable——主机不可达	 	||x|
|3	|2	|Protocol Unreachable——协议不可达	| 	|x|
|3	|3	|Port Unreachable——端口不可达	 	||x|
|3	|4	|Fragmentation needed but no frag. bit set——需要进行分片但设置不分片比特	 |	|x|
|3	|5	|Source routing failed——源站选路失败	| 	|x|
|3	|6	|Destination network unknown——目的网络未知	|| 	x|
|3	|7	|Destination host unknown——目的主机未知	 ||	x|
|3	|9	|Destination network administratively prohibited——目的网络被强制禁止	 ||	x|
|3	|10	|Destination host administratively prohibited——目的主机被强制禁止	|| 	x|
|3	|11	|Network unreachable for TOS——由于服务类型TOS，网络不可达	|| 	x|
|3	|12	|Host unreachable for TOS——由于服务类型TOS，主机不可达	 ||	x|
|3	|13	|Communication administratively prohibited by filtering——由于过滤，通信被强制禁止	|| 	x|
|3	|14	|Host precedence violation——主机越权	|| 	x|
|3	|15	|Precedence cutoff in effect——优先中止生效	|| 	x|
|4	|0	|Source quench——源端被关闭（基本流控制）	||| 	 
|5	|0	|Redirect for network——对网络重定向	 |||	 
|5	|1	|Redirect for host——对主机重定向	||| 	 
|5	|2	|Redirect for TOS and network——对服务类型和网络重定向	 |||	 
|5	|3	|Redirect for TOS and host——对服务类型和主机重定向	 |||	 
|8	|0	|Echo request——回显请求（Ping请求）	|x	 ||
|9	|0	|Router advertisement——路由器通告	||| 	 
|10	|0	|Route solicitation——路由器请求	 |||	 
|11	|0	|TTL equals 0 during transit——传输期间生存时间为0	|| 	x|
|11	|1	|TTL equals 0 during reassembly——在数据报组装期间生存时间为0	|| 	x|
|12	|0	|IP header bad (catchall error)——坏的IP首部（包括各种差错）	|| 	x|
|12	|1	|Required options missing——缺少必需的选项	|| 	x|
|17	|0	|Address mask request——地址掩码请求	|x	 ||
|18	|0	|Address mask reply——地址掩码应答 |||

# 应用
## ping
ping 命令执行的时候，源主机首先会构建一个 ICMP 请求数据包，ICMP 数据包内包含多个字段。最重要的是两个，第一个是类型字段，对于请求数据包而言该字段为 8；另外一个是顺序号，主要用于区分连续 ping 的时候发出的多个数据包。每发出一个请求数据包，顺序号会自动加 1。为了能够计算往返时间 RTT，它会在报文的数据部分插入发送时间。

然后，由 ICMP 协议将这个数据包连同地址 192.168.1.2 一起交给 IP 层。IP 层将以 192.168.1.2 作为目的地址，本机 IP 地址作为源地址，加上一些其他控制信息，构建一个 IP 数据包。

接下来，需要加入 MAC 头。如果在本节 ARP 映射表中查找出 IP 地址 192.168.1.2 所对应的 MAC 地址，则可以直接使用；如果没有，则需要发送 ARP 协议查询 MAC 地址，获得 MAC 地址后，由数据链路层构建一个数据帧，目的地址是 IP 层传过来的 MAC 地址，源地址则是本机的 MAC 地址；还要附加上一些控制信息，依据以太网的介质访问规则，将它们传送出去。

主机 B 收到这个数据帧后，先检查它的目的 MAC 地址，并和本机的 MAC 地址对比，如符合，则接收，否则就丢弃。接收后检查该数据帧，将 IP 数据包从帧中提取出来，交给本机的 IP 层。同样，IP 层检查后，将有用的信息提取后交给 ICMP 协议。

主机 B 会构建一个 ICMP 应答包，应答数据包的类型字段为 0，顺序号为接收到的请求数据包中的顺序号，然后再发送出去给主机 A。

在规定的时候间内，源主机如果没有接到 ICMP 的应答包，则说明目标主机不可达；如果接收到了 ICMP 应答包，则说明目标主机可达。此时，源主机会检查，用当前时刻减去该数据包最初从源主机上发出的时刻，就是 ICMP 数据包的时间延迟。

## traceroute
traceroute是用来检测发出数据包的主机到目标主机之间所经过的网关数量的工具。traceroute的原理是试图以最小的TTL（存活时间）发出探测包来跟踪数据包到达目标主机所经过的网关，然后监听一个来自网关ICMP的应答。发送数据包的大小默认为38个字节。

原理：程序利用增加存活时间（TTL）来实现其功能。每当数据包(3个数据包包括源地址，目的地址和包发出的时间标签)经过一个路由器，其存活时间就会减1。当其存活时间是0时，主机便取消数据包，并传送一个ICMP（Internet控制报文协议。它是TCP/IP协议族的一个子协议，用于在IP主机、路由器之间传递控制消息。控制消息是指网络通不通、主机是否可达、路由是否可用等网络本身的消息。这些控制消息虽然并不传输用户数据，但是对于用户数据的传递起着重要的作用。） TTL数据包给原数据包的发出者。

traceroute程序完整过程：首先它发送一份TTL字段为1的IP数据包给目的主机，处理这个数据包的第一个路由器将TTL值减1，然后丢弃该数据报，并给源主机发送一个ICMP报文（“超时”信息，这个报文包含了路由器的IP地址，这样就得到了第一个路由器的地址），然后traceroute发送一个TTL为2的数据报来得到第二个路由器的IP地址，继续这个过程，直至这个数据报到达目的主机。

# 参考资料
[ICMP与ping](https://time.geekbang.org/column/article/8445)
[Linux命令：traceroute命令（路由跟踪）](https://blog.csdn.net/sinat_33442459/article/details/75126149)