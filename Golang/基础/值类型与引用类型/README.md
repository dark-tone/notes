golang 数据类型按 **存储方式** 有两大类数据类型：
- 值类型，也叫基本数据类型：数值类型、bool、string、数组、struct 结构体
- 引用数据类型：指针、slice 切片、管道 chan、map、以及 interface

通常是<br>
值类型 =》栈中分配<br>
引用类型 =》堆上分配<br>
但是因为逃逸分析 可能会发生改变

# new 和 make区别
二者都是用来做内存分配的。

make只用于slice、map以及channel的初始化，返回的是类型本身（类型本身就是引用类型（指针类型））；

而new用于内存分配时，在内存中存储的是对应类型的型零值（比如0，false），返回的是该类型的指针类型。

# 参考资料
[golang 中的数据类型](https://cloud.tencent.com/developer/article/1936057)

[new与make的区别](https://juejin.cn/post/7131717990558466062)