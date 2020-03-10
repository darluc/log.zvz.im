title: Go：Goroutine 的抢占机制
date: 2020-3-11 01:16:02
tags:
- golang
---

![](https://img.zvz.im/imgs/2020/03/f1eb2d0c781c6429.png)

ℹ︎*本文内容基于 Go 1.13 版本。*

Go 语言利用内部的调度器管理 goroutine。这个调度器致力于对 goroutine 进行切换，确保它们都能够获得执行时间。不过，调度器有时会抢占这些 goroutine 的运行时间以保证正确的轮换。

<!-- more -->

## 调度器和抢占机制

让我们使用一个简单的例子来说明调度器是如何工作的：*为了便于阅读，这些例子不会使用原子操作*。

```go
func main() {
   var total int
   var wg sync.WaitGroup

   for i := 0; i < 10; i++ {
      wg.Add(1)
      go func() {
         for j := 0; j < 1000; j++ {
            total += readNumber()
         }
         wg.Done()
      }()
   }

   wg.Wait()
}

//go:noinline
func readNumber() int {
   return rand.Intn(10)
}
```

下面是追踪信息（tracing）：

![](https://img.zvz.im/imgs/2020/03/c99454113005d9e4.png)

我们可以清楚地观察到调度器控制 goroutine 在处理器上进行轮换，对它们全部给予相应的执行时间。当 goroutine 由于系统调用，channel 阻塞，睡眠，等待互斥量等操作而停止时，Go 会对其进行调度。在上一个例子中，调度器利用了数字生成器中的互斥量，从而给所有 goroutine 执行时间。在追踪信息中也是可以看到的：

![](https://img.zvz.im/imgs/2020/03/2642990de8ce33d0.png)

不过，如果 goroutine 自身没有任何的停滞，Go 还是需要有办法停止正在运行的 goroutine。这种行为被称作抢占（preemption），它允许调度器对 goroutine 的执行进行切换。任何执行时间超过 10 毫秒的 goroutine 会被标记为可抢占。而后，抢占行为会发生在函数调用的开始阶段，goroutine 调用栈增加的时候。

让我们来看一个例子，它与上个例子的区别在于去除了数字生成器中的锁：

```go
func main() {
   var total int
   var wg sync.WaitGroup

   for i := gen(0); i < 20; i++ {
      wg.Add(1)
      go func(g gen) {
         for j := 0; j < 1e7; j++ {
            total += g.readNumber()
         }
         wg.Done()
      }(i)
   }

   wg.Wait()
}

var generators [20]*rand.Rand

func init() {
   for i := int64(0); i < 20; i++  {
      generators[i] = rand.New(rand.NewSource(i).(rand.Source64))
   }
}

type gen int
//go:noinline
func (g gen) readNumber() int {
   return generators[int(g)].Intn(10)
}
```

这里是追踪信息：

![](https://img.zvz.im/imgs/2020/03/99b3bfeed8081050.png)

而且，goroutine 是在函数调用开始阶段被抢占的：

![](https://img.zvz.im/imgs/2020/03/07e3cba823a8e131.png)

这个检查过程是由编译器自动加入的；这里有一段上例生成的汇编代码：

![](https://img.zvz.im/imgs/2020/03/eb10dd47a8eb65ed.png)

通过将指令插入在每个函数执行前，这个 runtime 调用确保栈可以增加。同时使得调度器在必要时可以运行。

绝大多数情况下，goroutine 都会给调度器对它们进行调度的能力。但是，一个没有函数调用的循环却可以阻塞调度。

## 强制抢占

我们从一个简单的例子开始展示一个循环是如何阻塞调度过程的：

```go
func main() {
   var total int
   var wg sync.WaitGroup

   for i := 0; i < 20; i++ {
      wg.Add(1)
      go func() {
         for j := 0; j < 1e6; j++ {
            total ++
         }
         wg.Done()
      }()
   }

   wg.Wait()
}
```

由于没有调用函数，所以这些 goroutine 永远不会被阻塞，调度器也无法进行抢占。我们在追踪信息中可以看到：

![](https://img.zvz.im/imgs/2020/03/069ad543fdbb429e.png)

<center><small>Goroutine 无法被抢占</small></center>

不过，Go 语言提供了几种解决方案来修复此问题：

* 使用 `runtime.Gosched()` 强制调度器执行

```go
for j := 0; j < 1e8; j++ {
   if j % 1e7 == 0 {
      runtime.Gosched()
   }
   total ++
}
```

新的追踪信息如下：

![](https://img.zvz.im/imgs/2020/03/5da58bc015fec606.png)

* 使用实验特性让循环可以被抢占。要启用这个特性，可以使用指令 `GOEXPERIMENT=preemptibleloops` 重新编译 Go 工具链，或者在 `go build` 时增加参数 `-gcflags -d=ssa/insert_resched_checks/on` 。这次就无需修改代码；以下是新的追踪信息：

![](https://img.zvz.im/imgs/2020/03/10c5c18af9634270.png)

当循环中的抢占被启用时，编译器会在生成 SSA 代码时加入一段过程。

![](https://img.zvz.im/imgs/2020/03/ea0c3522f02f71c9.png)

这一段过程会添加一些指令时不时地调用一下调度器：

![](https://img.zvz.im/imgs/2020/03/7e2c3bd1d9bead3c.png)

*想要了解更多关于 Go 编译器的内容，我建议你阅读我的文章「Go: 编译器概览」。*

然而，这种方式会降低一点代码的执行速度，因为它强制调用调度器的次数，可能比实际需要的更多。这里有两个版本的跑分比较：

```shell
name    old time/op  new time/op  delta
Loop-8   2.18s ± 2%   2.05s ± 1%  -6.23%
```

## 即将到来的改进

到目前为止，调度器采用的是可以覆盖绝大多数情况的合作式抢占技术。不过，在某些特殊的场景下，它可能成为一个痛点。一个关于「非合作式抢占」的新提案已经提交，它着眼于解决文档中说明的此类问题：

> 我提议将 Go 的实现切换为非合作式抢占，它应该允许 goroutine 大体上在任意执行点可以被抢占，而不需要显示地进行抢占检查。这种方式可以解决抢占延迟的问题而且不占用运行时开支。

这份文档给出了几种建议的优缺点，并且可能会在下个版本的 Go 中落地。





翻译自：[Go: Goroutine and Preemption](https://medium.com/a-journey-with-go/go-goroutine-and-preemption-d6bc2aa2f4b7)

> 译者注：Go 1.14 版本目前已经改变了 goroutine 的抢占方式。具体可参考 [Go 1.14 Release Notes](https://golang.org/doc/go1.14#runtime)

