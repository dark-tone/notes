# 环境变量
## GOROOT
Go 语言安装根目录的路径，也就是 GO 语言的安装路径。

## GOPATH
存放sdk以外的第三方类库或自己收藏的可复用的代码，一般含有src、pkg、bin子目录。
- src存放源代码(比如：.go .c .h .s等) 按照golang默认约定，go run，go install等命令的当前工作路径（即在此路径下执行上述命令）。
- pkg编译时生成的中间文件（比如：.a）　　golang编译包时
- bin编译后生成的可执行文件

## GOBIN
GO 程序生成的可执行文件（executable file）的路径。
