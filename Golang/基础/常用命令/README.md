# go get
下载导入路径指定的包及其依赖项，然后安装命名包，即执行go install命令。

用法：go get [-d] [-f] [-t] [-u] [-fix] [-insecure] [build flags] [packages]

-d 让命令程序只执行下载动作，而不执行安装动作。<br>
-f 仅在使用-u标记时才有效。该标记会让命令程序忽略掉对已下载代码包的导入路径的检查。如果下载并安装的代码包所属的项目是你从别人那里Fork过来的，那么这样做就尤为重要了。<br>
-fix 让命令程序在下载代码包后先执行修正动作，而后再进行编译和安装。<br>
-insecure 允许命令程序使用非安全的scheme（如HTTP）去下载指定的代码包。如果你用的代码仓库（如公司内部的Gitlab）没有HTTPS支持，可以添加此标记。请在确定安全的情况下使用它。<br>
-t 让命令程序同时下载并安装指定的代码包中的测试源码文件中依赖的代码包。<br>
**-u** 让命令利用网络来更新已有代码包及其依赖包。默认情况下，该命令只会从网络上下载本地不存在的代码包，而不会更新已有的代码包。<br>
```
// 将会升级到最新的次要版本或者修订版本(x.y.z, z是修订版本号， y是次要版本号)
go get -u package

// get指定版本的依赖包
go get package[@version]
```

# go install
和go build命令比较相似，go build命令会编译包及其依赖，生成的文件存放在当前目录下。而且go build只对main包有效，其他包不起作用。而go install对于非main包会生成静态文件放在$GOPATH/pkg目录下，文件扩展名为a。如果为main包，则会在$GOPATH/bin下生成一个和给定包名相同的可执行二进制文件。

# go mod
**go mod init** 初始化当前⽂件夹, 创建go.mod⽂件。<br>
**go mod tidy** 增加缺少的module，删除未使用的module。<br>
**go mod download** 下载依赖的module到本地cache（默认为$GOPATH/pkg/mod⽬录）

# go build
编译构建生成文件。

# go run
编译并运行程序。

# go vet
检查 Go 源代码并报告可疑的情况

# go test
用于运行测试用例<br>
**-cover**： 同时输出代码覆盖率

# go tool compile -S
输出对应代码的汇编代码
``` go
go tool compile -S xxx.go > xxx.s
```