title: 日知其所无：修复 go get 私有仓库时返回的 410 错误
date: 2021-01-30 22:17:59
tags:

- golang

---

![](https://img.zvz.im/imgs/2021/01/a36a3a4d605ce3a9.png)

今天原本应该是个好日子，令人兴致盎然的星期天，却被一件让我抓狂了一整个小时的事情毁掉了。其实，从前一晚开始（周六晚）我就被这件事难住了，不过我实在太累了，所以我决定暂停休息一下，周日才继续研究。

我一直投入时间在做 Marbar ([https://mabar.id](https://mabar.id/)) 这个业余项目。目前它还处于 beta 版本状态，仍然缺少许多特性，你可以在 Android Play Store [这里](http://bit.ly/2mcWvv0)下载体验一下。

我个人对与 Mabar 这个项目有很多的期待，它将会是在各种界面（手机，桌面，网页）上都能使用的一个平台。技术栈方面，我使用 Golang 作为后端语言，Kubernetes 作为基础设施，还有 Digital Ocean 提供的服务器。

## 问题

问题是这样的，我有一个私有模块（Golang 模块），是后端 API 引入的一个简单类库。但是无论如何，我都无法获取到这个依赖包，而且当我执行 `go get` 命令时总是会报错。
<!-- more -->
假设这个包的名称是 `lucifer`。它总是会在命令行中显示如下错误信息。太让我抓狂了。

```bash
$ go get -v bitbucket.org/compay/lucifer
go: finding bitbucket.org/compay/lucifer latest
go: downloading bitbucket.org/compay/lucifer v0.0.0-20190921175342-61a76c096369
verifying bitbucket.org/compay/lucifer@v0.0.0-20190921175342-61a76c096369: bitbucket.org/compay/lucifer@v0.0.0-20190921175342-61a76c096369: reading https://sum.golang.org/lookup/bitbucket.org/compay/lucifer@v0.0.0-20190921175342-61a76c096369: 410 Gone
```

如果你阅读一下这些信息，它在说这个包在 sum.golang.org 中已经不存在或者不可用了。

首先，我想到的是可能忘记对 bitbucket 强制使用 SSH 了，因为我之前有写过相关的问题：https://medium.com/easyread/today-i-learned-fix-go-get-private-repository-return-error-terminal-prompts-disabled-8c5549d89045

但是，它仍然无法工作。即使我强制它只许使用 SSH，当我使用 `go get` 命令它还是继续报错。

## 根本原因

然后我在互联网上搜索了这个问题后，我找到了真正的原因。这个问题只会出现在 Golang 版本 1.13 之后。在看过发布文档 https://golang.org/doc/go1.13#modules 之后我更加确信了。

这个问题之所以会发生是因为这个版本的 Golang 有一个新的特性。

## 解决方案

实际上我们有几种可供选择的解决方案。

* **使用 GOPRIVATE**

  如 Go 1.13 的发布文档所述：

  > *新的 GOPRIVATE 环境变量个用于指示非公开可见的模块路径。并且会作为底层 GONOPROXY 和 GONOSUMDB 变量的默认值，这两个变量则对于哪些模块通过代理获取并且校验 checksum 数据库提供更加细化的控制。*

  也就是说，要解决上面的问题，我们只需在系统中设置 `GOPRIVATE` 变量。在 `~/.bashrc` 中加入这行指令。*请根据你的公司和组织名称调整命令内容*

  ```bash
  export GOPRIVATE="gitlab.com/idmabar,bitbucket.org/idmabar,github.com/idmabar"
  ```

  要确认它是否已经生效，你可以使用 `go env` 命令。命令回显如下：

  ```shell
  $ go env
  GO111MODULE=""
  GOARCH="amd64"
  GOBIN=""
  GOCACHE="/Users/imantumorang/Library/Caches/go-build"
  GOENV="/Users/imantumorang/Library/Application Support/go/env"
  GOEXE=""
  GOFLAGS=""
  GOHOSTARCH="amd64"
  GOHOSTOS="darwin"
  GOOS="darwin"
  GOPATH="/Users/imantumorang/go"
  GOPRIVATE="gitlab.com/idmabar,bitbucket.org/idmabar,github.com/idmabar"
  GOPROXY="https://proxy.golang.org,direct"
  GOROOT="/usr/local/Cellar/go/1.13/libexec"
  GOSUMDB="sum.golang.org"
  GOTMPDIR=""
  GOTOOLDIR="/usr/local/Cellar/go/1.13/libexec/pkg/tool/darwin_amd64"
  GCCGO="gccgo"
  AR="ar"
  CC="clang"
  CXX="clang++"
  CGO_ENABLED="1"
  GOMOD=""
  CGO_CFLAGS="-g -O2"
  CGO_CPPFLAGS=""
  CGO_CXXFLAGS="-g -O2"
  CGO_FFLAGS="-g -O2"
  CGO_LDFLAGS="-g -O2"
  ```

  现在我可以使用 `go get` 命令获取我的私有库了。

  ```shell
  $ go get bitbucket.org/company/lucifer
  go: finding bitbucket.org/company/lucifer latest
  go: downloading bitbucket.org/company/lucifer v0.0.0-20190921175342-61a76c096369
  go: extracting bitbucket.org/company/lucifer v0.0.0-20190921175342-61a76c096369
  ```

  所以这个 `环境变量` 会告诉 `go get` 命令使用私有主机代理去获取代码包。

* 使用 GONOSUMDB

  另一个解决方案是可以使用 `GONOSUMDB` 变量。我还没有试过，不过按这份提议所说应该是可行的 https://go.googlesource.com/proposal/+/master/design/25530-sumdb.md

  你可以这样设定你的环境变量：

  ```shell
  export GONOSUMDB="gitlab.com/idmabar,bitbucket.org/idmabar,github.com/idmabar"
  ```

  实际上这个问题只会发生在最新的 Golang 1.13 以及其后的版本。所以在升级你的 Golang 版本之前，请确认你设定了这个环境变量。

  这里有一些与此问题相关的链接，多亏 [noveaustack](https://stackoverflow.com/users/12052086/noveaustack) 发现这个问题并且发到了 Stackoverflow 上，我只是又发了一遍而已，因为我刚刚了解到这个问题并且使其成为了我的新知。

## 参考

* Stackoverflow 回答：https://stackoverflow.com/a/57887036/4075313
* go 模块的 Go Sum DB 提案：https://go.googlesource.com/proposal/+/master/design/25530-sumdb.md
* 代理 Checksum DB：https://docs.gomods.io/configuration/sumdb/
* 相关的 Github 事项：[#33985](https://github.com/golang/go/issues/33985) 还有 [#32291](https://github.com/golang/go/issues/32291)







翻译自：[Today I Learned — Fix: go get private repository return error reading sum.golang.org/lookup … 410 gone](https://medium.com/mabar/today-i-learned-fix-go-get-private-repository-return-error-reading-sum-golang-org-lookup-93058a058dd8)

