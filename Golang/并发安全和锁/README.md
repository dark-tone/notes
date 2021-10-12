# 并发安全和锁
## 互斥锁
互斥锁是一种常用的控制共享资源访问的方法，它能够保证同时只有一个goroutine可以访问共享资源。Go语言中使用sync包的Mutex类型来实现互斥锁。语法如下：
```
func main() {
    var lock sync.Mutex
    lock.Lock() // 加锁
    .... // do something
    lock.Unlock() // 解锁
}
```

## 读写互斥锁
读写锁分为两种：读锁和写锁。当一个goroutine获取读锁之后，其他的goroutine如果是获取读锁会继续获得锁，如果是获取写锁就会等待；当一个goroutine获取写锁之后，其他的goroutine无论是获取读锁还是写锁都会等待。
```
var rwLock sync.RWMutex
rwlock.Lock() // 加写锁
rwlock.Unlock() // 解写锁
rwlock.RLock() // 加读锁
rwlock.RUnlock() // 解读锁
```
RWMutex只有当获得锁的大部分goroutine都是读操作，而锁在竞争条件下，也就是说，goroutine们必须等待才能获取到锁的时候，RWMutex才是最能带来好处的。RWMutex需要更复杂的内部记录，所以会让它比一般的无竞争锁的mutex慢一些。

## sync.WaitGroup
sync.WaitGroup内部维护着一个计数器，计数器的值可以增加和减少。例如当我们启动了N 个并发任务时，就将计数器值增加N。每个任务完成时通过调用Done()方法将计数器减1。通过调用Wait()来等待并发任务执行完，当计数器值为0时，表示所有并发任务已经完成。

包含的三个方法：
```
(wg * WaitGroup) Add(delta int)	计数器+delta
(wg *WaitGroup) Done()	计数器-1
(wg *WaitGroup) Wait()	阻塞直到计数器变为0
```
示例：
```
var wg sync.WaitGroup

func hello() {
	defer wg.Done()
	fmt.Println("Hello Goroutine!")
}
func main() {
	wg.Add(1)
	go hello() // 启动另外一个goroutine去执行hello函数
	fmt.Println("main goroutine done!")
	wg.Wait()
}
```

## sync.Once
在编程的很多场景下我们需要确保某些操作在高并发的场景下只执行一次，例如只加载一次配置文件、只关闭一次通道等。

Go语言中的sync包中提供了一个针对只执行一次场景的解决方案sync.Once。

sync.Once只有一个Do方法，其签名如下：
```
func (o *Once) Do(f func()) {}
```
加载图标示例：
```
var loadIconsOnce sync.Once
var icons map[string]image.Image
// Concurrency-safe.
func Icon(name string) image.Image {
    loadIconsOnce.Do(loadIcons)
    return icons[name]
}
```

## sync.Map
Go语言中内置的map不是并发安全的。Go语言的sync包中提供了一个开箱即用的并发安全版map sync.Map。开箱即用表示不用像内置的map一样使用make函数初始化就能直接使用。同时sync.Map内置了诸如Store、Load、LoadOrStore、Delete、Range等操作方法。

# 参考文章
《Go语言圣经》

[Go语言基础之并发](https://www.liwenzhou.com/posts/Go/14_concurrence/)