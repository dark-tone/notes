# 指针类型
## 普通指针
(*T)，用于传递对象地址，不能进行指针运算，属于**类型安全指针**。
> 类型安全指针的限制：
>1. 两个任意指针类型不能随意转换。
>2. 不能对指针的地址进行算术运算。

## unsafe.Pointer
单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算。是普通指针与uintptr之间转换的桥梁。

## uintptr
可用于指针运算的，GC不把uintptr当指针，也就是说uintptr无法持有对象，也就无法阻止GC回收uintptr对应的内存地址的内存数据。

# unsafe包函数
## Offsetof
返回**结构体成员**在内存中的位置离结构体起始处(结构体的第一个字段的偏移量都是0)的字节数，即偏移量，通过源码注释可知其入参必须是一个结构体。

## Sizeof
返回变量在内存中占用的字节数(切记，如果是slice，则不会返回这个slice在内存中的实际占用长度)。
> 对于和平台有关的int类型，这个要看平台是32位还是64位，会取最大的。

## Alignof
Alignof返回一个类型的对齐值，也可以叫做对齐系数或者对齐倍数。对齐值是一个和内存对齐有关的值，合理的内存对齐可以提高内存读写的性能
> 对齐值一般是2^n,最大不会超过8，需要注意的是每个人的电脑运行的结果可能不一样。

# 内存对齐
CPU 读取内存是一块一块读取的，块的大小可以为 2、4、6、8、16 字节等大小。块大小我们称其为内存访问粒度。

## 为什么要做对齐
- 平台（移植性）原因：不是所有的硬件平台都能够访问任意地址上的任意数据。例如：特定的硬件平台只允许在特定地址获取特定类型的数据，否则会导致异常情况.
- 性能原因：若访问未对齐的内存，将会导致 CPU 进行两次内存访问，并且要花费额外的时钟周期来处理对齐及运算。而本身就对齐的内存仅需要一次访问就可以完成读取动作。

## 对齐规则
- 对于**具体类型**来说，对齐值=min(编译器默认对齐值，类型大小Sizeof长度)。也就是在默认设置的对齐值和类型的内存占用大小之间。
    > 对于结构体的成员变量，对齐后还会有偏移量，**其偏移量必须为对齐值的整数倍**。
- **struct在每个字段都内存对齐之后，其本身也要进行对齐**，对齐值=min(默认对齐值，字段最大类型长度)。这条也很好理解，struct的所有字段中，最大的那个类型的长度以及默认对齐值之间，取最小的那个。
    > 结构体是占用一块连续的内存，由于有对齐规则，所以**struct字段顺序不同，最终大小可能不同**。

例子：
``` go
type Bar1 struct {
	x int32 // 4
	y int64  // 8
	z bool  // 1
}
var b1 Bar1
fmt.Println(unsafe.Sizeof(b1)) // 24

type Bar2 struct {
	x int32 // 4
	z bool  // 1
    y int64  // 8
}
var b2 Bar2
fmt.Println(unsafe.Sizeof(b2)) // 16
```

# 常用场景
## 利用 Pointer 作为中介，完成 T1 类型 到 T2 类型的转换
T1 和 T2 是任意类型，如果 T1 的内存占用大于等于 T2，并且 T1 和 T2 的内存布局一致，可以利用 Pointer 作为中介，完成 T1类型 到 T2类型的转换。（如果T1 的内存占用小于 T2，那么 T2 剩余部分没法赋值，就会有问题）
``` go
func main() {
    // slice 转 string，可以正确转换
    sli := []byte{'a', 'b', 'c'}
    str := *(*string)(unsafe.Pointer(&sli))
    fmt.Println(str)      // abc
    fmt.Println(len(str)) // 3

    // string 转 slice，cap 字段无法赋值，无法正确转换
    str = "1234"
    b := *(*[]byte)(unsafe.Pointer(&str))
    fmt.Println(string(b)) // 1234
    fmt.Println(len(b))    // 4
    fmt.Println(cap(b))    // 824634066744
}
```

## string与[]byte的无拷贝转换
``` go
// 字节切片转 string
func ByteSlice2String(slice []byte) (s string) {
    sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&slice))
    stringHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))
    stringHeader.Data = sliceHeader.Data
    stringHeader.Len = sliceHeader.Len
    return
}

// string 转字节切片
func String2ByteSlice(s string) (slice []byte) {
    stringHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))
    sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&slice))

    sliceHeader.Data = stringHeader.Data
    sliceHeader.Len = stringHeader.Len
    sliceHeader.Cap = stringHeader.Len
    return
}

func main() {
    b := []byte{'h', 'e', 'l', 'l', 'o'}
    fmt.Println(ByteSlice2String(b)) // hello

    s := "hello"
    fmt.Println(String2ByteSlice(s)) // [104 101 108 108 111]
  
}  
```
**由于默认字符串内存是分配在不可修改区的，使用上述的 String2ByteSlice将 string 转为 slice 后，只能进行读取，不能修改其底层数据值**。


# 参考资料
[golang中的内存对齐和unsafe初探](https://zhuanlan.zhihu.com/p/460965095)

[go之unsafe](https://www.jianshu.com/p/b0736d252e9a)

[Go结构体的内存对齐](https://zhuanlan.zhihu.com/p/389128696)

[Go语言unsafe.Pointer浅析](https://www.jianshu.com/p/e3fa8aac8168)