title: Go 语言基础 - 编写单元测试
date: 2019-03-09 01:23:45
tags:
- golang
- testing
---

在上一篇文章 ["Grab JSON from an API"](http://blog.alexellis.io/golang-json-api-client/) 中，我们探索了如何使用 HTTP 客户端以及如何解析 JSON 数据。本篇文章是 Go 语言主题的续篇，讲述如何编写单元测试。

## 1. Go 语言中的测试

Go 语言有一个自带的测试命令 `go test` ，还有一个标准 `testing` 测试包，它能够为你提供一个小却完整的测试体验。

这套标准工具链还包括了基准测试以及基于语句的代码覆盖率测试，类似与 NCover(.Net) 或者 Istanbul(Node.js)。

## 1.2 编写测试代码

和 Go 语言其它方面如格式化、命名规则一样，Go 语言的单元测试也显得个性十足。它的语法刻意规避了使用断言模式，并将值验证和行为检测的工作留给了开发人员。
<!-- more -->

这儿有一个例子，我们要对 `main` 包里的一个方法进行测试。我们已定义了一个名为 `Sum` 的出口函数，它接收两个整数参数，并将它们相加。

```go
package main

func Sum(x int, y int) int {
    return x + y
}

func main() {
    Sum(5, 5)
}
```

我们在另一个单独的文件中编写测试代码。这个测试文件可以在其它的包（目录）中，或者在相同的包中（`main`）。以下是一个检测相加结果的单元测试：

```go
package main

import "testing"

func TestSum(t *testing.T) {
    total := Sum(5, 5)
    if total != 10 {
       t.Errorf("Sum was incorrect, got: %d, want: %d.", total, 10)
    }
}
```

Go 语言的测试函数有以下特征：

* 只有唯一的参数，必须是 `t *testing.T` 类型
* 必须以单词 `Test` 开头，再组合上首字母大写的单词或词组（一般是被测试的方法名称，如 `TestValidateClient`）
* 调用 `t.Error` 或者 `t.Fail` 方法指明测试失败（这里我使用了 `t.Errorf` 来提供更多的细节）
* `t.Log` 可以用来提供一些失败信息以外的调试信息
* 测试代码文件名必须是 `_test` 结尾的形式 `something_test.go` ，例如：`addtion_test.go`

> 如果你在同一个目录下既有代码也有测试代码，那么你就无法使用 `go run *.go` 的方式执行你的程序了。我一般会使用 `go build` 编译出可执行程序，再执行它。

你可能更习惯于使用 `Assert` 关键字进行验证工作，不过 [The Go Programming Language](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440) 的作者们对于 Go 的断言方式做了许多很好的辩解。

当使用断言时：

* 测试代码往往会让人觉得他们正在使用另一种语言（比如 RSpec/Mocha）
* 错误输出看起来令人费解 "assert: 0 == 1"
* 可能会产生大量的调用栈信息
* 第一个断言失败后，测试代码会终止执行 - 会掩盖其它的失败可能

> 有一些类似 RSpec 或者 Assert 的 Go 语言第三方测试库。比如 [stretchr/testify](https://github.com/stretchr/testify)。

### 测试表

“测试表”的概念是一组测试输入和输出值的映射。这是一个针对 `Sum` 函数的例子：

```go
package main

import "testing"

func TestSum(t *testing.T) {
	tables := []struct {
		x int
		y int
		n int
	}{
		{1, 1, 2},
		{1, 2, 3},
		{2, 2, 4},
		{5, 2, 7},
	}

	for _, table := range tables {
		total := Sum(table.x, table.y)
		if total != table.n {
			t.Errorf("Sum of (%d+%d) was incorrect, got: %d, want: %d.", table.x, table.y, total, table.n)
		}
	}
}
```

如果你想要制造一些错误使得测试无法通过，那么将 `Sum` 函数的返回部分改为 `x * y` 即可。

```shell
$ go test -v
=== RUN   TestSum
--- FAIL: TestSum (0.00s)
	table_test.go:19: Sum of (1+1) was incorrect, got: 1, want: 2.
	table_test.go:19: Sum of (1+2) was incorrect, got: 2, want: 3.
	table_test.go:19: Sum of (5+2) was incorrect, got: 10, want: 7.
FAIL
exit status 1
FAIL	github.com/alexellis/t6	0.013s
```

### 启动测试

有两种方式可以用来启动一个包内的测试代码。这些方法对于单元测试和集成测试是相同的。

1. 在和测试文件相同的目录中：

   ```shell
   go test
   ```

   这会执行包内所有匹配 _test.go 名称的测试代码

   或者

2. 采用完整的包名

   ```shell
   go test github.com/alexellis/golangbasics1
   ```

现在你可以执行 Go 语言单元测试了，可以使用 `go test -v ` 获得更详细的输出，你能看到每条测试的 PASS/FAIL 信息，以及所有 `t.Log` 打印出的额外日志信息。

> 单元测试和集成测试的区别在于，单元测试通常独立于外部依赖，不会与网络、磁盘等产生交互。单元测试一般只关注函数的功能。

## 1.3 `go test` 的更多用法

### 语句（statement）覆盖率

`go test` 工具自带内建的代码语句覆盖率测试功能。想要用之前的代码例子尝试一下，输入以下命令即可：

```shell
$ go test -cover
PASS
coverage: 50.0% of statements
ok  	github.com/alexellis/golangbasics1	0.009s
```

较高的语句覆盖率比低覆盖率或者零覆盖率要好，不过这样量化也可能会产生误导。我们想保证我们不只是在执行语句，而且我们还验证了代码的行为和输出，而且在不符合逻辑的地方报错。如果你删除了之前例子代码中的 “if” 语句，它仍然会保持 50% 的测试覆盖率，却丧失了验证 “Sum” 方法行为的用处。

### 生成 HTML 格式的覆盖率测试报告

如果你使用接下来的两条命令，你就可以直观地看到你的程序哪些部分被覆盖到了，而哪些语句没有被覆盖到：

```shell
go test -cover -coverprofile=c.out
go tool cover -html=c.out -o coverage.html 
```

然后用浏览器打开 coverage.html 文件。

### Go 编译时不会引入你的测试代码

还有一点，将 `addition_test.go` 这样的测试文件留在你的包目录中虽然略有些不自然。不过 Go 语言的编译器和链接器保证不会将你的测试文件编入任何它生成的二进制文件中。

下面有个例子，可以找出 net/http 包中的生成代码和测试代码。

```shell
$ go list -f={{.GoFiles}} net/http
[client.go cookie.go doc.go filetransport.go fs.go h2_bundle.go header.go http.go jar.go method.go request.go response.go server.go sniff.go status.go transfer.go transport.go]

$ go list -f={{.TestGoFiles}} net/http
[cookie_test.go export_test.go filetransport_test.go header_test.go http_test.go proxy_test.go range_test.go readrequest_test.go requestwrite_test.go response_test.go responsewrite_test.go transfer_test.go transport_internal_test.go]
```

想要了解更多的基础内容可以阅读 [Golang testing docs](https://golang.org/pkg/testing/)。

## 1.4 脱离依赖

定义单元测试概念的关键点就是，它能够脱离运行时的依赖项或合作者。

这在 Go 语言中是通过接口来实现的，不过如果你有 C# 或者 Java 的背景，它们的接口看起来和 Go 会有些许不同。Go 语言中接口是隐含的，而不是一种强制措施。意味着实际的类并不需要知道接口的存在。

这意味着我们可以定义非常多的小接口，如 [io.ReadCloser](https://golang.org/src/io/io.go?s=4977:5022#L116) 它只包含两个方法分别来自于 Reader 和 Closer 接口：

```go
Read(p []byte) (n int, err error)
```

*Reader* 接口

```go
Close() error
```

*Closer* 接口

如果你在设计一个会被第三方使用的包，那么定义适当的接口就会显得非常有意义，因为其他人需要时，可以利用这些接口让单元测试代码能够不依赖于你的代码包。

接口的具体实现在函数调用时可以被替换。如果我们想要测试这个方法，我们可以提供一个实现了 Reader 接口的伪造类。

```go
package main

import (
	"fmt"
	"io"
)

type FakeReader struct {
}

func (FakeReader) Read(p []byte) (n int, err error) {
	// return an integer and error or nil
}

func ReadAllTheBytes(reader io.Reader) []byte {
	// read from the reader..
}

func main() {
	fakeReader := FakeReader{}
	// You could create a method called SetFakeBytes which initialises canned data.
	fakeReader.SetFakeBytes([]byte("when called, return this data"))
	bytes := ReadAllTheBytes(fakeReader)
	fmt.Printf("%d bytes read.\n", len(bytes))
}
```

在实现你自己的抽象前，去 Golang 文档中搜索一下是否已有现成可用的东西，总会是个不错的主意。对于上面的例子我们也可以使用标准库中的 [bytes](https://golang.org/pkg/bytes/) 包：

```go
func NewReader(b []byte) *Reader
```

Go 语言的 `testing/iotest` 包提供了一些 Reader 的实现类，有些执行起来比较慢，有些会在读数据的中途产生错误。这些实现对于适应性测试都非常好用。

* Golang 文档：[testing/iotest](https://golang.org/pkg/testing/iotest/)

## 1.5 工作示例

接下来我要重构[上一篇文章](http://blog.alexellis.io/golang-json-api-client/)中寻找宇宙中有多少宇航员的示例代码。

让我们从测试代码开始：

```go
package main

import "testing"

type testWebRequest struct {
}

func (testWebRequest) FetchBytes(url string) []byte {
	return []byte(`{"number": 2}`)
}

func TestGetAstronauts(t *testing.T) {
	amount := GetAstronauts(testWebRequest{})
	if amount != 1 {
		t.Errorf("People in space, got: %d, want: %d.", amount, 1)
	}
}
```

这里我有一个名为 GetAstronauts 的外部方法，它从一个 HTTP 终端读取字节信息，解析成一个结构体，最后返回 “number” 属性的整数值。

我伪造的方法只返回了能满足测试的最小化的 JSON 信息，我先让它返回与判断不符的数字，以便我能确定测试代码工作正常。因为很难保证第一次执行就通过的测试代码，是否真起了作用。

这是程序代码中的 `main` 方法。`GetAstronauts` 方法用一个接口作为它的第一个参数，使得我们能够独立于此代码中的任何 HTTP 逻辑，以及它的引入依赖项。

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

func GetAstronauts(getWebRequest GetWebRequest) int {
	url := "http://api.open-notify.org/astros.json"
	bodyBytes := getWebRequest.FetchBytes(url)
	peopleResult := people{}
	jsonErr := json.Unmarshal(bodyBytes, &peopleResult)
	if jsonErr != nil {
		log.Fatal(jsonErr)
	}
	return peopleResult.Number
}

func main() {
	liveClient := LiveGetWebRequest{}
	number := GetAstronauts(liveClient)

	fmt.Println(number)
}
```

GetWebRequest 接口包含了以下方法：

```go
type GetWebRequest interface {
	FetchBytes(url string) []byte
}
```

> 接口是推断出来的，而不是显示声明的。这一点与 C# 和 Java 语言不同。

完整的 types.go 文件代码如下，它是从前一篇博文中截取出来的：

```go
package main

import (
	"io/ioutil"
	"log"
	"net/http"
	"time"
)

type people struct {
	Number int `json:"number"`
}

type GetWebRequest interface {
	FetchBytes(url string) []byte
}

type LiveGetWebRequest struct {
}

func (LiveGetWebRequest) FetchBytes(url string) []byte {
	spaceClient := http.Client{
		Timeout: time.Second * 2, // Maximum of 2 secs
	}

	req, err := http.NewRequest(http.MethodGet, url, nil)
	if err != nil {
		log.Fatal(err)
	}

	req.Header.Set("User-Agent", "spacecount-tutorial")

	res, getErr := spaceClient.Do(req)
	if getErr != nil {
		log.Fatal(getErr)
	}

	body, readErr := ioutil.ReadAll(res.Body)
	if readErr != nil {
		log.Fatal(readErr)
	}
	return body
}
```

### 选择抽象的对象

上面的单元测试仅有效地测试了 `json.Unmarshal` 方法以及我们假想的正常 HTTP 响应结果。这种抽象对于我们的例子来说没有问题，不过代码测试覆盖率比较低。

我们还可以再做一些底层测试，来保证 HTTP 请求的强制2秒超时是否正确，或者我们创建一个 GET 请求而不是 POST 请求。

令人高兴的是，Go 语言自带了一组用来伪造 HTTP 服务端和客户端的工具方法。

### 更进一步：

* 探索 [http/httptest 包](https://golang.org/pkg/net/http/httptest/#pkg-examples)
* 使用伪造的 HTTP 客户端重构上面的测试代码
* 重构之前和之后的测试覆盖率分别是多少？



翻译自：[Golang basics - writing unit tests](https://blog.alexellis.io/golang-writing-unit-tests/)