零拷贝并不是没有拷贝数据，而是减少用户态、内核态的切换次数以及CPU拷贝次数；实现零拷贝主要有三种方式分别是
1. mmap + write
2. sendfile
3. 带有DMA收集拷贝功能的sendfile

# 参考资料
[什么是零拷贝](https://blog.csdn.net/weixin_39406430/article/details/123715072)

[一文搞懂零拷贝实现原理与使用(图解)](https://zhuanlan.zhihu.com/p/485632980)