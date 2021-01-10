title: Go 语言中的 init 函数
date: 2021-01-10 01:18:16
tags:

- golang
---

标识符 `main` 无所不在。每个 Go 程序的执行都是从 `main` 包中一个拥有相同名字的函数开始的。当这个 `main` 函数返回时，整个程序也退出了执行。`init` 函数也扮演着特定的角色，本文会描述它们的特性并介绍它们的使用方法。

`init` 函数是定义在包级别的，它被用于：

* 初始化无法使用表达式初始化的变量
* 检查和修复程序的状态
* 注册
* 执行一次性的运算
* 以及其它

除了下面要介绍一些区别，你可以将任何在一般函数中有效的代码放在其中。

## 包的初始化

要使用一个引入的包，首先它需要被初始化。这是由 Golang 的运行系统来完成的，由以下几步（顺序很重要）组成：

1. 初始化引入的包（递归释义）
2. 计算并初始化赋值包级别的变量
3. 执行包内的 `init` 方法

> *包的初始化过程只会被执行一次，即使它被多次引用*
<!--more-->

## 顺序

Go 语言的包可以包含许多文件。那么在这些包和文件中，变量的初始化和 `init` 函数的执行顺序是怎样的呢？首先，初始化依赖机制会起作用（详情可以查看[“Go 中的初始化依赖”](https://medium.com/golangspec/initialization-dependencies-in-go-51ae7b53f24c)）。当依赖工作完成后，必须决定先初始化 *a.go* 文件中的变量还是 *z.go* 文件中的变量。这依赖于文件在编译器中出现的顺序。如果 *z.go* 先被提交给构建系统，那么它的变量就会先于 *a.go* 中的变量初始化。`init` 方法的调用也遵守相同的顺序。语言规格定义中建议总是采用相同的顺序，并且将包中的文件按单词拼写顺序传入：

> 为了保证初始化行为可稳定复现，构建系统应该倾向于将同一个包中的多个文件按文件名的单词拼写顺序传递给编译器。

不过对于移植性较差的程序来说也可以使用特别的顺序。我们用下面的例子看看这些是如何一起工作的：

**sandbox.go**

```go
package main
import "fmt"
var _ int64 = s()
func init() {
    fmt.Println("init in sandbox.go")
}
func s() int64 {
    fmt.Println("calling s() in sandbox.go")
    return 1
}
func main() {
    fmt.Println("main")
}
```

**a.go**

```go
package main
import "fmt"
var _ int64 = a()
func init() {
    fmt.Println("init in a.go")
}
func a() int64 {
    fmt.Println("calling a() in a.go")
    return 2
}
```

**z.go**

```go
package main
import "fmt"
var _ int64 = z()
func init() {
    fmt.Println("init in z.go")
}
func z() int64 {
    fmt.Println("calling z() in z.go")
    return 3
}
```

程序输出：

```shell
calling a() in a.go
calling s() in sandbox.go
calling z() in z.go
init in a.go
init in sandbox.go
init in z.go
main
```

## 属性

`init` 函数不接受任何参数，也没有返回值。于 `main` 相比，标识符 `init` 是没有被申明的，所以无法被引用：

```go
package main
import "fmt"
func init() {
    fmt.Println("init")
}
func main() {
    init()
}
```

编译时它会输出 “undefined: init” 错误。

> 正式地来讲，*init* 标识符不会引入任何绑定关系。与此相同的还有，下划线表示的空白标识符。

同一个包或文件中可以定义许多的 *init* 函数：

**sandbox.go**

```go
package main
import "fmt"
func init() {
    fmt.Println("init 1")
}
func init() {
    fmt.Println("init 2")
}
func main() {
    fmt.Println("main")
}
```

**utils.go**

```go
package main
import "fmt"
func init() {
    fmt.Println("init 3")
}
```

输出如下：

```shell
init 1
init 2
init 3
main
```

> init 函数在标准库中被频繁地使用，比如：在[math](https://github.com/golang/go/blob/2878cf14f3bb4c097771e50a481fec43962d7401/src/math/pow10.go#L33)，[bzip2](https://github.com/golang/go/blob/2878cf14f3bb4c097771e50a481fec43962d7401/src/compress/bzip2/bzip2.go#L479) 和 [image](https://github.com/golang/go/blob/2d573eee8ae532a3720ef4efbff9c8f42b6e8217/src/image/gif/reader.go#L511) 这些包里。

*init* 函数最常见的使用场景就是赋值无法用初始化表达式计算得出的情况：

```go
var precomputed = [20]float64{}
func init() {
    var current float64 = 1
    precomputed[0] = current
    for i := 1; i < len(precomputed); i++ {
        precomputed[i] = precomputed[i-1] * 1.2
    }
}
```

*for* 循环无法用作[表达式](https://golang.org/ref/spec#Expression)，所以将其放入 *init* 函数中可以解决这个问题。

## 为副作用而引入包

Go 对于未使用的包引入非常严格。有时候程序员引入一个包可能只是为了执行其中的 *init* 函数进行初始化工作。空白标识符这时候就派上用场了：

```go
import _ "image/png"
```

这在 [image](https://github.com/golang/go/blob/0104a31b8fbcbe52728a08867b26415d282c35d2/src/image/image.go#L10) 包的评论中都有被提到。

如果上面介绍的这些内容有帮助到你，请关注我的博客以获取最新的更新，同时也可以提升我的积极性。





翻译自：[init functions in Go](https://medium.com/golangspec/init-functions-in-go-eac191b3860a)

