# Timer
``` go
func main() {
    // 创建后即开始计时（放入go框架维护的一个timer堆中，超时后会gc）
    timer := time.NewTimer(time.Second * 5)
    // 会等待5s
    <- timer.C
    fmt.Println("wai 5s")

    // 使用time.After的写法
    select {
    case <-time.After(time.Second * 5):
        fmt.Println("xxx")
    }
}
```

# Ticker
``` go
func main() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    for range ticker.C {
        fmt.Println("xxx")
}
```
有一个defer去停止ticker，为什么要这么做呢？创建定时器就是把定时器的runtimeTimer放到由维护协程维护的堆中，一次性定时器到期后，会从堆中删除，如果没有到期则调用Stop方法实现删除。但是，周期性定时器是不会执行删除动作的，所以如果项目里面持续创建多个周期性定时器并没有stop的话，会导致堆越来越大，从而引起资源泄露。


# 参考资料
[Go中的定时器](https://www.cnblogs.com/zhangcaiwang/p/15130774.html)

[Go定时器--Timer](https://www.cnblogs.com/failymao/p/15064059.html)