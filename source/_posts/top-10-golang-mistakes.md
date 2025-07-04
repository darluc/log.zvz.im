title: Go 项目最常见的 10 种错误
date: 2019-11-09 16:59:25
tags:
- golang
---

![](https://img.zvz.im/imgs/2019/10/19578038f26ae531.jpg)

本文是我总结的 golang 项目中最常见的十种错误。排名不分先后。

## 未知状态的枚举值

我们来看一个简单的例子：

```go
type Status uint32

const (
	StatusOpen Status = iota
	StatusClosed
	StatusUnknown
)
```

在这里我们使用 `iota`  创建了一个枚举列表，结果状态如下：

```go
StatusOpen = 0
StatusClosed = 1
StatusUnknown = 2
```

现在，假设这个 `Status` 类型是 JSON 请求数据中的一部分，并且会被用于编码解码。我们可以设计一个结构，如下：

```go
type Request struct {
	ID        int    `json:"Id"`
	Timestamp int    `json:"Timestamp"`
	Status    Status `json:"Status"`
}
```
<!--more-->

然后，接收到这样的请求数据：

```json
{
  "Id": 1234,
  "Timestamp": 1563362390,
  "Status": 0
}
```

这儿没有什么特别的，状态字段会被解码为 `StatusOpen`，没错吧？但是，当我们使用另一个没有设置状态值的请求（无论是什么原因造成的）：

```json
{
  "Id": 1235,
  "Timestamp": 1563362390
}
```

这种情况下，`Request` 结构体的 `Status` 字段会被初始化为**零值**（`uint32` 类型：0）。所以，它的值会是 `StatusOpen` 而不是 `StatusUnknown`。

最好的办法是将一个枚举的未知状态值设为 0：

```go
type Status uint32

const (
	StatusUnknown Status = iota
	StatusOpen
	StatusClosed
)
```

这样一来，当 JSON 请求的缺少状态字段的时候，它就会像我们所预期地那样初始化为 `StatusUnknown` 。

## 性能测试

想要准确地进行性能测试是很困难的。有许多因素能影响到测试的结果。

一个常见的错误是你会被某些编译器优化所愚弄。我们看一个来自于 [_teivah/bitvector_](https://github.com/teivah/bitvector/) 库中的实际例子：

```go
func clear(n uint64, i, j uint8) uint64 {
	return (math.MaxUint64<<j | ((1 << i) - 1)) & n
}
```

此方法将指定范围的二进制位进行清除。要测试它的性能，我们也许会这样做：

```go
func BenchmarkWrong(b *testing.B) {
	for i := 0; i < b.N; i++ {
		clear(1221892080809121, 10, 63)
	}
}
```

在这个测试中，编译器会注意到 `clear` 是一个叶子方法（没有调用任何其它的方法），所以会对它进行内联编译。它被内联编译后，编译器会注意到这段代码没有任何**副作用**（side-effects）。所以 `clear` 的调用过程会被简单地移除掉，最终导致结果不正确。

一个可行的方法是将计算结果赋值给一个全局变量：

```go
var result uint64

func BenchmarkCorrect(b *testing.B) {
	var r uint64
	for i := 0; i < b.N; i++ {
		r = clear(1221892080809121, 10, 63)
	}
	result = r
}
```

这样，编译器就无法知道该方法调用是否会产生副作用了。因此，这个基准测试就变得准确了。

## 指针！到处乱用指针！

变量以值传递时会产生这个变量的拷贝，而通过指针传递时只会拷贝内存地址。

所以，指针传递总是**更快**一些，不是嘛？

如果你是这样认为的，那么请看一下[这个例子](https://gist.github.com/teivah/a32a8e9039314a48f03538f3f9535537)。这是一个使用了 0.3KB 大小的数据结构，进行指针传递和值传递比较的性能测试。0.3KB 并不大，和我们（大多数人）日常见到的数据结构的大小差不多。

我在本地环境运行这些性能测试时，值传递竟然比指针传递**快 4 倍**。这真的有些反印象流，不是吗？

对这种结果的解释是，它与 Go 的内存管理相关。我没法像 [William Kennedy](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html) 解释得那样清楚，不过还是尽量总结一下。

一个变量可能被分配在**堆**上或者**栈**上。粗略的说明如下：

* 栈内保存着 **goroutine** 中**生存着**的变量。一旦方法返回，这些变量就会被从栈中抛弃。
* 堆里保存着**共享的**变量（比如， 全局变量）

我们来看一个简单的例子，它会返回一个值：

```go
func getFooValue() foo {
	var result foo
	// Do something
	return result
}
```

代码中，一个 `result` 变量由当前的 goroutine 创建。这个变量被放入当前栈内。一旦方法返回，调用者会接收到这个变量的拷贝。而这个变量自身则会被从栈内抛出。它仍然存在于内存中，直到它被别的变量抹除，同时它是**无法访问**的。

然后，同样的例子返回一个指针：

```go
func getFooPointer() *foo {
	var result foo
	// Do something
	return &result
}
```

这个 `result` 变量仍然由当前的 goroutine 创建，不过调用者会接收到一个指针（变量地址的拷贝）。假如 `result` 变量被从栈内抛出，那么方法调用者也将**无法访问**到它了。

在这种场景下，Go 编译器会让 `result` 变量**逃逸（escape）**到一个存放共享变量的地方：**堆**。

传递指针参数，则是另一个场景了。比如：

```go
func main()  {
	p := &foo{}
	f(p)
}
```

由于我们在同一个 goroutine 中调用 `f` 方法，所以变量 `p` 不需要逃逸。只需简单地将它压入栈中，子方法就能够访问到。

比如，这直接导致 `io.Reader` 接口的 `Read` 方法被设计成接收一个 slice 参数而不是返回一个 slice。返回一个 slice（slice 是指针类型）会使它被转移到堆中去。

为什么栈的访问速度**快**呢？主要有两个原因：

* 栈不需要**垃圾回收器**。正如之前所述，变量在创建时入栈，方法返回时出栈。没有必要实现一个复杂流程来回收未使用的变量等等。
* 每个 goroutine 都有一个栈，所以在栈内存放变量和堆内不同，不需要**同步**机制。这也会使得效率提升。

总结一下，当我们构建方法时，我们应该默认使用**使用值而不是指针**。指针应当只在我们需要**共享**变量时使用。

所以，假如我们遇到了性能问题，一个可能的优化方案就是检查某些场景下是否该使用指针。如果想要知道编译器何时将变量转移到堆中，我们可以使用指令：`go build -gcflags "-m -m"` 。

再强调一下，对于日常使用的大多数场景，传值是最好的选择。

## 跳出 for/switch 或 for/select 嵌套

下面例子中的 `f` 如果返回 true 会发生什么呢？

```go

for {
  switch f() {
  case true:
    break
  case false:
    // Do something
  }
}
```

我们会调用 `break` 语句。不过，它会从 `switch`  语句中跳出，而**不是 for 循环**。

类似的情况：

```go
for {
  select {
  case <-ch:
  // Do something
  case <-ctx.Done():
    break
  }
}
```

`break` 语句与 `select` 控制相关，而不是 for 循环。

一种办法是使用**带标签的 break** 从 for/switch 或者 for/select 嵌套中跳出：

```go
loop:
	for {
		select {
		case <-ch:
		// Do something
		case <-ctx.Done():
			break loop
		}
	}
```

## 错误管理

Go 语言还在不断发展之中，目前对于错误的处理方式还稍显稚嫩。不出意外的话，错误管理会是 Go 2 版本中最受期待的特性。

当前版本（Go 1.13 之前）的标准库只提供了一些错误的创建方法，你可以看看 [_pkg/errors](https://github.com/pkg/errors) 包。

遵守以下经验法则使用该库是一个比较好的方式，但并非总是如此：

> 一个错误应当只进行**一次**处理。记录错误日志**也是**对错误的一种处理。所以一个错误**只能**被记录**或者**抛出。

使用当前版本的标准库，很难遵守上面的准则，因为我们会想给一个错误增加一些上下文信息，从而形成某种形式的继承关系。

让我们举一个例子，在一次 REST 请求产生 DB 报错时，可能希望有如下返回：

```
unable to serve HTTP POST request for customer 1234
 |_ unable to insert customer contract abcd
     |_ unable to commit transaction
```

我们使用 *pkg/errors* 库时可能会这样做：

```go
func postHandler(customer Customer) Status {
	err := insert(customer.Contract)
	if err != nil {
		log.WithError(err).Errorf("unable to serve HTTP POST request for customer %s", customer.ID)
		return Status{ok: false}
	}
	return Status{ok: true}
}

func insert(contract Contract) error {
	err := dbQuery(contract)
	if err != nil {
		return errors.Wrapf(err, "unable to insert customer contract %s", contract.ID)
	}
	return nil
}

func dbQuery(contract Contract) error {
	// Do something then fail
	return errors.New("unable to commit transaction")
}
```

初始的错误（如果不是由外部库返回的）可能是用 `errors.New` 创建的。中间一层，`insert` ，对这个错误加入了更多的上下文信息。随后，上一层处理了这个错误，并将其记录了下来。每个层级的逻辑都处理了这个错误或将其抛出。

我们可能还想检查一下错误的起因，比如用来实现重试机制。比如我们有一个外部的用于访问 数据库的 `db` 包。这个库可能会返回一个短暂的（临时的）错误叫作 `db.DBError`。为了判断我们是否需要进行重试，我们必须要确认错误的起因：

```go
func postHandler(customer Customer) Status {
	err := insert(customer.Contract)
	if err != nil {
		switch errors.Cause(err).(type) {
		default:
			log.WithError(err).Errorf("unable to serve HTTP POST request for customer %s", customer.ID)
			return Status{ok: false}
		case *db.DBError:
			return retry(customer)
		}

	}
	return Status{ok: true}
}

func insert(contract Contract) error {
	err := db.dbQuery(contract)
	if err != nil {
		return errors.Wrapf(err, "unable to insert customer contract %s", contract.ID)
	}
	return nil
}
```

这是通过 *pkg/errors* 包中的  `errors.Cause` 来实完成的：

我见过的一个常见错误是部分使用 *pkg/errors* 。检查错误时是这样做的：

```go
switch err.(type) {
default:
  log.WithError(err).Errorf("unable to serve HTTP POST request for customer %s", customer.ID)
  return Status{ok: false}
case *db.DBError:
  return retry(customer)
}
```

这个例子里，如果 `db.DBError` 被包裹后，就绝对无法触发重试机制了。

## Slice 初始化

有些时候，我们知道一个 slice 的最终长度。比如我们想将切片 `Foo` 转化为切片 `Bar` ，可知两个切片的长度相同。

我经常见到这样的初始化代码：

```go
var bars []Bar
bars := make([]Bar, 0)
```

切片并不是什么神奇的结构。它在底层实现了一个**扩容**策略，用于应对空间不足的情况。空间不足时，它会自动创建出一个新的数组（[容量更大](https://www.darkcoding.net/software/go-how-slices-grow/)）然后将现有的数据项拷贝进去。

现在，想象一下在 `[]Foo` 包含上万的元素时，我们如何多次重复这种扩容操作？虽然，对于一次插入分摊开的时间复杂度（平均）仍然是 O(1)，但实际上， 这样做会产生**性能影响**。

所以，如果我们知道切片的最终长度，就可以采用以下的手段：

* 将它初始化为一个预定义的长度：

  ```go
  func convert(foos []Foo) []Bar {
  	bars := make([]Bar, len(foos))
  	for i, foo := range foos {
  		bars[i] = fooToBar(foo)
  	}
  	return bars
  }
  ```

* 或者初始化为预定义的容量，但长度等于零：

  ```go
  func convert(foos []Foo) []Bar {
  	bars := make([]Bar, 0, len(foos))
  	for _, foo := range foos {
  		bars = append(bars, fooToBar(foo))
  	}
  	return bars
  }
  ```


哪一种办法更好？第一种执行速度快一些。不过，你或许更喜欢第二种方式，因为它具有更好的一致性：无论我们是否知道初始的大小，都可以使用 `append` 在切片末尾加入新的元素。

## 上下文管理

`context.Context` 常常会被开发者误解。官方文档描述如下：

> *一个 Context 包含一个生存期限，一个撤销信号，以及其它跨越 API 边界的数据*。

这个描述足以使得一些人感到困惑，为什么以及如何使用它。

我们试着解释得更详细一些。一个上下文包含：

* 一个**生存期限**。它可以是一个时间长度（比如，250 毫秒），或者一个时间点（比如，`2019-01-08 01:00:00`），当到达截止时间的时候，我们必须取消某个正在运行的活动（比如，一个正在等待 channel 输入的 I/O 请求）。
* 一个**撤销信号**（基本上就是一个 `<-chan struct{}`）。这里的行为也是相似的。一旦我们接受到信号，我们必须终止一个正在运行的活动。举例来说，假设我们有两个请求。一个要插入一些数据，而另一个是撤销第一个请求（因为它没有意义或者其它别的原因）。这里可以使用第一个请求的可撤销上下文（ cancelable context ）来达成目的，一旦我们接收到第二个请求就执行撤销动作。
* 一组键值对（都是 `interface{}` 类型）。

再补充两点。第一，上下文对象是可组装的。一个上下文可以带有一个生存期限和一组键值对。而且，多个 Go 协程可以**共享**同一个上下文，因此一个撤销信号有能力终止**多个活动**。

回到我们的主题上来，这里有一个我见过的真实错误案例。

有一个基于 [*urfave/cli*](https://github.com/urfave/cli) （以防你不知道，这是一个很好的用来创建命令行程序的 Go 代码库）的 Go 程序。程序一开始，研发人员就使用了库中一个类似程序上下文的东西。 程序终止时，会用这个上下文会发送一个撤销信号。

我遇到的情况是这个上下文被直接传入到对另一个服务端的 gRPC 调用中。而我们并**不想**这样做。

相反地，我们想告诉 gRPC 库：*请在应用程序终止或者请求超过 100 毫秒后，取消请求*。

为了实现这种效果，我们只要创建一个合成的上下文。如果 `parent` 是那个应用上下文（由 `urfave/cli` 创建），那么我们只要这样做：

```go
ctx, cancel := context.WithTimeout(parent, 100 * time.Millisecond)
response, err := grpcClient.Send(ctx, request)
```

上下文并没有复杂到无法理解的程度，而且我认为它是 Go 语言最好的特性之一。

## 不使用 -race 参数

我确实经常见到的一个错误是，测试 Go 应用时不使用 `-race` 参数。

就如这份[报告](https://zvz.im/2019/07/25/real-world-concurrency-bugs-in-golang/)中描述的，尽管 Go 的设计初衷是“让并发编程变得简单可靠”，我们仍然会遇到许多并发问题。

当然 Go 的竞争探测器（ race detector ）不可能帮你解决所有的并发问题。不过，它仍然是很**有价值的**工具，当进行程序测试时我们总是应该启用它。

## 使用文件名作为入参

另一个普遍的问题是将文件名作为方法的入参。

假设我们要实现一个功能对文件中的空行进行计数。最自然的实现方式大概是这样的：

```go
func count(filename string) (int, error) {
	file, err := os.Open(filename)
	if err != nil {
		return 0, errors.Wrapf(err, "unable to open %s", filename)
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	count := 0
	for scanner.Scan() {
		if scanner.Text() == "" {
			count++
		}
	}
	return count, nil
}
```

文件名 `filename` 作为参数，然后我们打开这个文件执行逻辑，没错吧？

现在，假设我们要对这个函数进行**单元测试**，使用一个正常的文件，一个空文件，一个采用不同编码的文件，等等。它会很快就变得难以管理。

而且，如果想对一个 HTTP 的请求体执行相同的逻辑，我们就不得不另外写一个函数了。

Go 自带两个很棒的接口抽象：`io.Reader` 和 `io.Writer` 。我们可以简单地使用一个 `io.Reader`   **抽象**数据源代替文件名作为入参。

它是一个文件？一个 HTTP 请求体？或是一个字节缓冲区？这都不重要，因为我们会使用相同的 `Read` 方法。

在这个案例中，我们甚至可以将输入缓冲起来逐行读入。这样，我们就可以使用 `bufio.Reader` 和它的 `ReadLine` 方法。

```go
func count(reader *bufio.Reader) (int, error) {
	count := 0
	for {
		line, _, err := reader.ReadLine()
		if err != nil {
			switch err {
			default:
				return 0, errors.Wrapf(err, "unable to read")
			case io.EOF:
				return count, nil
			}
		}
		if len(line) == 0 {
			count++
		}
	}
}
```

打开文件的任务就交给 `count` 函数的使用者了。

```go
file, err := os.Open(filename)
if err != nil {
  return errors.Wrapf(err, "unable to open %s", filename)
}
defer file.Close()
count, err := count(bufio.NewReader(file))
```

采用第二种实现方式，函数可以**无视**数据源的真实载体进行调用。同时，这也便于我们实现单元测试，因为我们只需使用一个 `string` 字符串创建一个 `bufio.Reader` 就可以了。

```go
count, err := count(bufio.NewReader(strings.NewReader("input")))
```

## Go 协程和循环中的变量

最后一个常见错误是在协程中使用循环内的变量。

下面这个例子的输出会是什么？

```go
ints := []int{1, 2, 3}
for _, i := range ints {
  go func() {
    fmt.Printf("%v\n", i)
  }()
}
```

`1 2 3` 以任意顺序出现？不对。

在这个例子中，所有的协程**共用**了同一个变量实例，所以它会打印出 `3 3 3`（最可能）。

这个问题有两种解决方案。第一种是将变量 `i` 的值传递到闭包中（内嵌函数）：

```go
ints := []int{1, 2, 3}
for _, i := range ints {
  go func(i int) {
    fmt.Printf("%v\n", i)
  }(i)
}
```

第二种方法是在 for 循环中创建另一个变量：

```go
ints := []int{1, 2, 3}
for _, i := range ints {
  i := i
  go func() {
    fmt.Printf("%v\n", i)
  }()
}
```

采用 `i := i` 的方式看起可能有些奇怪，却是完全有效的。在循环体内就意味着在另一个作用域中。所以 `i := i` 创建了另一个名为 `i` 的变量实例。当然，我们可以为了提高可读性而采用另一个名字。





> 翻译自：[The Top 10 Most Common Mistakes I’ve Seen in Go Projects](top-10-most-common-mistakes-ive-seen-in-go-projects-4b79d4f6cd65)