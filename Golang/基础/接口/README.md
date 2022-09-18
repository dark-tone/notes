# 接口
接口类型是对其它类型行为的抽象和概括；因为接口类型不会和特定的实现细节绑定在一起，通过这种抽象的方式我们可以让我们的函数更加灵活和更具有适应能力。

## 接口内嵌
```
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}

// 使用内嵌
type ReadWriter interface {
    Reader
    Writer
}

// 不使用内嵌
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

## 类型断言
针对interface{}，一个接口的值（简称接口值）是由一个具体类型和具体类型的值两部分组成的。这两部分分别称为接口的**动态类型**和**动态值**。

### 语法格式
```
x.(T)
```
