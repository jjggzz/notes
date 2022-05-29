# 简介和helloword

## 简介

1. go=c+python，即go语言既能拥有c的运行速度，又能像python一样快速开发。
2. go从c语言中继承了很多理念（表达式语法、控制结构、基础数据类型、指针等等）
3. 引入了包的概念，每个文件都要归属于一个包
4. 自动垃圾回收
5. 天然并发
6. 函数可以返回多个值

## helloworld

新建hello.go文件，内容如下

```go
//每个文件都要属于一个包
package main

//导入fmt库，可以使用fmt的函数
import "fmt"

//主函数入口，golang中所有的 { 都要跟在语句的末尾，否则报错
func main() {
    //打印hello world
    fmt.Println("hello word")
}
```

  通过命令编译该文件，会生成可执行文件

```shell
go build -o hello hello.go
```

  执行生成的文件

```she
./hello
```

