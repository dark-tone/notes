# 简介
golang性能调试工具，两种用法，导入runtime/pprof包或者导入net/http/pprof包，这里记录一下后者的用法。

# 用法
## 简易版
```
package main

import (
    "net/http"
    _ "net/http/pprof"
)
func main() {
    http.ListenAndServe("0.0.0.0:6060", nil)
}
```
原理：引入net/http/pprof后，会自动执行其init()方法，init()方法会给http的DefaultServeMux注册相关接口。

## 自定义
```
package main
import (
	"net/http"
	"net/http/pprof"
)
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/debug/pprof/", pprof.Index)
	mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
	mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
	mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
	http.ListenAndServe(":8081", mux)
}
```

# 常用功能
可访问"[host]:[port]/debug/pprof"查看可使用的功能。
- allocs：查看过去所有内存分配的样本。
- block：查看导致阻塞同步的堆栈跟踪。
- cmdline： 当前程序的命令行的完整调用路径。
- goroutine：查看当前所有运行的 goroutines 堆栈跟踪。
- heap：查看活动对象的内存分配情况。
- mutex：查看导致互斥锁的竞争持有者的堆栈跟踪。
- profile： 默认进行 30s 的 CPU Profiling，得到一个分析用的 profile 文件。
- threadcreate：查看创建新 OS 线程的堆栈跟踪。

# 命令行模式
总的分析思路就是通过top 和traces 找出关键函数，再通过list <Func> 查看函数代码，找到关键代码行并确认优化方案。
## top [num]
查看占用最高的num项，输入 **-cum** 按cum进行排序。
- flat：可以理解为函数占用的CPU时间。
- cum：当前函数以及当前函数调用其他函数的时间（火焰图）。

## traces [regex]
打印匹配的调用栈。
查看匹配的函数具体每一行的CPU或内存等占用数据。


## list [func name]
查看匹配的函数具体每一行的CPU或内存等占用数据。

## 示例
1. 调用http://xxxxx/debug/pprof/heap 获取dump文件，存放路径假设：/download/heap
2. 命令行输入 <code>go tool pprof /download/heap</code> 进入pprof模式
3. 输入 <code>top 20 -cum</code> 查看前20占用资源最高的方法调用，寻找可疑函数funcX
4. 输入 <code>traces funcX</code> 查找函数调用栈，寻找可疑的函数funxY
5. 输入 <code>list funcY</code> 查看函数的具体位置

// 采样pprof信息并进入pprof命令行<br>
// go tool pprof -seconds=5 http://localhost:8080/debug/pprof/profile


# 可视化界面
需要先安装graphviz。
```
sudo apt install graphviz
```
## 命令
```
// 生成svg图片
go tool pprof -svg -output 123.svg http://127.0.0.1:8081/debug/pprof/profile

// 在localhost:8082启动web server
go tool pprof -http localhost:8082 http://127.0.0.1:8081/debug/pprof/heap
```

## Graph(函数逻辑调用树)
- 节点的颜色越红，其cum和cum%越大；其颜色越灰白，则cum和cum%越小
- 节点越大，其flat和flat%越大；其越小，则flat和flat%越小
- 线条代表了函数的调用链，线条越粗，代表指向的函数消耗了越多的资源
- 虚线连接表示该调用关系实际是经过了中间函数，但是这些（耗时少的）中间函数没有在最终的图中体现出来
- 带有inline字段表示该函数被内联进了调用方
- 如果图上的边或点太多导致看不清，可以使用--nodefraction和--edgefraction参数来隐藏占用比较小的边或点
> go tool pprof -http :9999 -edgefraction 0 -nodefraction 0 -nodecount 100000 cpu.prof

## Flame Graph即火焰图
- y轴表示调用方法的先后
- x轴表示在每个采样调用时间内，方法所占的时间/空间百分比
- 火焰图的横向长度表示cum，相比下面超出的一截代表flat，越宽代表占据cpu时间/mem大小越多

# 参考资料
[golang 性能分析(pprof)](https://blog.csdn.net/winter_wu_1998/article/details/104579701)
