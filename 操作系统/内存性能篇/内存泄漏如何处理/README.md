# 泄漏情况
- 栈内存由系统自动分配和管理。一旦程序运行超出了这个局部变量的作用域，栈内存就会被系统自动回收，所以不会产生内存泄漏的问题。
- 堆内存由应用程序自己来分配和管理。除非程序退出，这些堆内存并不会被系统自动释放，而是需要应用程序明确调用库函数 free() 来释放它们。如果应用程序没有正确释放堆内存，就会造成内存泄漏。
- 只读段，包括程序的代码和常量，由于是只读的，不会再去分配新的内存，所以也不会产生内存泄漏。
- 数据段，包括全局变量和静态变量，这些变量在定义时就已经确定了大小，所以也不会产生内存泄漏。
内存映射段，包括动态链接库和共享内存，其中共享内存由程序动态分配和管理。所以，如果程序在分配后忘了回收，就会导致跟堆内存类似的泄漏问题。

# memleak
memleak 是 bcc 软件包中的一个工具，执行 /usr/share/bcc/tools/memleak 就可以运行它。比如：
```
$ docker cp app:/app /app
$ /usr/share/bcc/tools/memleak -p $(pidof app) -a
Attaching to pid 12512, Ctrl+C to quit.
[03:00:41] Top 10 stacks with outstanding allocations:
    addr = 7f8f70863220 size = 8192
    addr = 7f8f70861210 size = 8192
    addr = 7f8f7085b1e0 size = 8192
    addr = 7f8f7085f200 size = 8192
    addr = 7f8f7085d1f0 size = 8192
    40960 bytes in 5 allocations from stack
        fibonacci+0x1f [app]
        child+0x4f [app]
        start_thread+0xdb [libpthread-2.27.so] 
```