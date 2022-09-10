# 常见内存泄漏
### goruntine泄漏
如goruntine内的channel一直在等待阻塞，或者陷入死循环等情况导致无法释放资源。

### time.Ticker未关闭导致泄漏
time.Ticker，必须调用stop方法结束，否则永远不会被释放。

# 预防/解决/检测
### channel及时关闭
如果是使用channel来通信的，必要时，接收端加上判断close的代码，当生产端逻辑结束后及时关闭channel。

### pprof
在出现内存泄漏时，使用pprof库能快速分析内存情况。