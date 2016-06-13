title: 让你的 Node.js 应用接口稳如狗：如何使用 Mocha, Chai 和 SuperTest 写测试代码
date: 2016-6-7 19:44:13
tags:
- Nodejs
- testing
---

你是否正在开发一套 Node.js 的 RESTful 接口，但又不确定如何进行终端测试呢？实际工作比你想象得要简单很多，如果你选择对了测试工具的话。

当我刚领到这个任务时：为我们的 node.js API 项目进行测试栈的调研和实现，我都不知道从哪儿开始。作为一个 ruby 码农，我之前都是用 RSpec 的。我的团队中没有人了解如何使用 javasript 对 RESTful 终端 API 进行测试。我们花了好几天来讨论该使用哪些测试库，包括 [Jasmine](http://jasmine.github.io/) 和 [Frisby](http://frisbyjs.com/)。

接下来就要介绍我们的最终解决方案了。我还加入了一些实现指南和一个 [GitHub 上的例子应用](https://github.com/kathrynhough/node-api-testing-example)，便于你将此测试技术栈加入到你自己的项目中去。
<!--more-->
## 测试技术栈

### Mocha

Mocha 是一个能让异步测试变得很简单的 javascript 测试框架。

**选用它的原因：**Mocha 既可以在 node.js  环境中运行，也可以运行在浏览器中。与其它的 javascript 测试框架比较起来，我们发现 Mocha 对于异步测试的处理是我们选择它的关键因素。当进行 API 测试时，我们会发送一些数据到终端，并且使用返回的数据向另一个终端发起调用请求。例如，我需要先获取到一个用户的数据，然后通过该用户的 id 获取到所有属于他的位置信息。

### Chai

与 Jasmine 不同，要让 Mocha 工作起来还需要额外的断言库。Chai 就是一个断言库，它可以让选择你最喜欢的接口，包括 “assert”，“expect” 还有 “should”。

**选用它的原因：**尽管 Mocha 可以与任意的断言库一起使用，Chai 也可以被任意的测试库所使用，但是许多 javascript 开发者还是选择将两者搭配使用。我们使用了 Chai 的 “expect” 接口将断言串联起来，这样我们就可以彻底的测试从 API 终端返回的 JSON 数据了。对于和我一样，有使用过 RSpec 的 ruby 程序员来说，Chai 非常容易上手，因为它的断言语法与 RSpec 的非常接近。

### SuperTest

SuperTest 是 SuperAgent 的一个扩展，一个轻量级的 HTTP AJAX 请求库。SuperTest 提供了用于测试 node.js API 终端响应的高阶抽象，使得断言便于理解。

**选用它的原因：**除了测验 JSON 对象的内容外，我们还想测验终端响应的其它数据，包括头信息类型和响应状态码。SuperTest 还有一个漂亮直观的接口。只需要简单地将接口终端需要的数据发送过去，就可以检验响应了。

### Magnum CI

Magnum CI 是一个可以为私有项目架设的持续集成服务。

**选用它的原因：**当我们的测试技术栈已基本完备时，还想在加入一个东西：那就是持续集成工具。我们的第一个目标是写好我们的测试代码，并使项目测试通过。接着，我们可以使用 Magnum CI，当我们每次推送代码到开发分支时，它会自动构建项目。Magnum CI 会执行我们的测试用例并报告成功或失败。我们喜欢 Magnum CI，因为它很容易搭建，可以和 [Bitbucket](https://bitbucket.org/) 集成，在我们白天工作的时候，会在项目的 Slack 频道中报告测试通过的情况。

既然我们已经选好了工具，那么立刻开始实现我们的测试解决方案吧。

## 如何开始

### 使用 npm 安装 Mocha

```shell
npm install -g mocha
```

这样会全局安装 Mocha。

### 使用 npm 安装 Chai

```shell
npm install chai
```

### 使用 npm 安装 SuperTest

```shell
npm install supertest
```

### 使用 npm 安装 Express

```shell
npm install express
```

### 使用 npm 安装 Body-Parser

```shell
npm install body-parser
```

### 设置你的测试目录

使用你的终端，进入你的项目所在的目录，然后用 *mkdir test* 指令新建一个测试目录。这个测试目录必须命名为 test ，这样 Mocha 才能找到所需执行的测试文件。

### 创建你的第一个文件

你可以给你的 mocha 文件起任意的名称。不过，如果你有成组的业务模型需要针对多个终端进行测试，那么我建议将它们命名为 "yourModel_test.js" 这种形式。这个例子里，就在 test 目录下新建一个 test.js 文件好了。

### 引用代码库

在测试文件开始，我们引入每一个要使用到的库，并将其赋值给一个全局变量。

```javascript
var should = require('chai').should(),
    expect = require('chai').expect,
    supertest = require('supertest'),
    api = supertest('http://localhost:3000');
```

### 设置 API URL

不要忘记将你的 API url 也赋值给一个全局变量。当你用 SuperTest 创建 RESTful 请求时，会用到这个变量。如果你有本地环境，可以将这个变量设置为 localhost。如果你想要在开发中进行测试，请保证这个 url 指向你的开发服务器地址。

### 第一个测试代码

如果你不太了解 BDD（Behavior-driven Development 行为驱动开发），请在继续读下去前，阅读 CodeShip 公司的伙计写的[这篇颇有帮助的文章](http://blog.codeship.com/behavior-driven-development/)。

简言之，行为驱动开发就是这样一种方法：将你置身于用户的位置考虑用户会期待应用如何响应，而不是作为开发者你想让它如何工作。不再去“写测试用例”，取而代之的是专注于指定应用的行为方式，让 mocha 保证你的应用能如你所期待地一样运行。因为我们在测试 API 接口，我们要保证每个终端都如设计的那样正确地运转，以正确的格式返回正确的数据。

我们的第一个测试，从一个 describe 函数开始，并传给它一个字符串类型的参数。这个字符串应该是一个名词，因为它说明了我们正在测试什么。

```javascript
describe('User', function(){
  
})
```

在这个 describe 方法中，我们将要写一系列的 it 方法，它们会详细描述我们想要测试的行为。我们给每个函数传递一段文本，使其能够明确地描述通过测试时期望的结果。这段文吧没有任何限制，但是保证它的可读性却很重要，因为当我们运行测试的时候，你将会在终端看到它。让我们开始完成第一个 it 方法。

```javascript
it('should return a 200 response', function(done){
  api.get('/users/1')
  .set('Accept', 'application/json')
  .expect(200, done);
})
```

执行一下。确保你处在你的项目的父级目录。然后执行 mocha 命令。

![](http://ww1.sinaimg.cn/large/7327fe71gw1f4ly0b3642j20gl06n3ys.jpg)

在此，我们简单地向一个服务端发送了一个 get 请求，并且期望它返回一个 200 状态的响应。

接下来，这儿有另一个测试，它出了检查 200 状态外，还会使用 Chai 的 expect 接口做一些额外的断言，因为相比 assert 和 should 接口我更喜欢 expect 一些。

```javascript
it('should be an object with keys and values', function(done){
  api.get('/users/1')
  .set('Accept', 'application/json')
  .expect(200)
  .end(function(err, res){
    expect(res.body).to.have.property('name');
    expect(res.body.name).to.not.equal(null);
    expect(res.body).to.have.property('email');
    expect(res.body.email).to.not.equal(null);
    expect(res.body).to.have.property('phoneNumber');
    expect(res.body.phoneNumber).to.not.equal(null);
    expect(res.body).to.have.property('role');
    expect(res.body.role).to.not.equal(null);
  })
})
```

让我们运行一下：

![](http://ww4.sinaimg.cn/large/7327fe71gw1f4lyd6qu79j20gm06n74p.jpg)

## 写更多的测试

### 回调函数

我们来写一个异步集成测试，先像 API 端发送一个 get 请求，然后使用得到的数据去更新记录。我们将使用回调函数来完成此事。Mocha 允许你使用 done() 回调函数，来发送信号给 Mocha，使其在转向下一个测试前等待当前测试完成。上一个例子中，我们使用 done() 来告诉 Mocha 先获取用户信息，现在我们要接着更新这个用户的信息。

```javascript
it('should be updated with a new name', function(done){
  api.put('/users/1')
  .set('Accept', 'application/x-www-form-urlencoded')
  .send({
    name: 'Kevin',
    email: 'kevin@example.com',
    phoneNumber: '999888777',
    role: 'editor'
  })
  .expect(200)
  .end(function(err, res){
    expect(res.body.name).to.equal('Kevin');
    expect(res.body.email).to.equal('kevin@example.com');
    expect(res.body.phoneNumber).to.equal('999888777');
    expect(res.body.role).to.equal('editor');
    done();
  })
})
```

让我们来运行一下：

![](http://ww4.sinaimg.cn/large/7327fe71gw1f4m0bad38rj20gp07w0t9.jpg)

### 错误

SuperTest 很管用不仅仅体现在测试你的API的 200 响应，还能应付错误响应和错误信息。我们来写个方法测试我们是否得到了正确的错误信息。

```javascript
it('should not be able to access other users locations', function(doen){
  api.get('/users/2/location')
  .set('Accept', 'application/x-www-form-urlencoded')
  .send({
    userId: 1
  })
  .expect(401)
  .end(function(err, res){
    if (err) return done(err);
    expect(res.error.text).to.equal('Unauthorized');
    done();
  })
})
```

看一下执行结果：

![](http://ww1.sinaimg.cn/large/7327fe71gw1f4m0vk3uduj20gl07cdgj.jpg)

### 钩子

有时候在集成测试中，你需要先创建一些记录然后再使用这些数据。只需要加入一个 before 方法你就可以很容易地达到这个目的。一定要把你的 before 方法写在 describe 方法内。现在，我们要创建一个 location 对象，这样就可以在测试用户 locations 服务时获取到 location 对象了。

```javascript
before(function(done){
  api.post('/locations')
  .set('Accept', 'application/x-www-form-urlencoded')
  .send({
    addressStreet: '111 Main St',
    addressCity: 'Portland',
    addressState: 'OR',
    addressZip: '97209',
    userId: 1
  })
  .expect('Content-Type', /json/)
  .expect(200)
  .end(function(err, res){
    location1 = res.body;
  })
})
```

```javascript
it('should access their own locations', function(done){
  api.get('/users/1/location')
  .set('Accept', 'application/x-www-form-urlencoded')
  .send({
    userId:1
  })
  .expect(200)
  .end(function(err, res){
    expect(res.body.userId).to.equal(1);
    expect(res.body.addresssCity).to.equal('Portland');
    done();
  })
})
```

看一下结果：

![](http://ww2.sinaimg.cn/large/7327fe71gw1f4mmt7lasmj20gq07lmxz.jpg)

除了 before 这个钩子外，mocha 还提供了 after，afterEach 和 beforeEach 这些钩子。

## 配置 Magnum CI

虽然我强烈推荐使用测试覆盖率工具[Istanbul](https://github.com/gotwarlost/istanbul)，但是我们还是决定等到项目的第二期在加入 Istanbul。因为现在的 node.js 接口还是比较简单的，我们决定采用抽查方式（就算我们只检测了200个响应）来保证每个接口都被测试过了。在我们确认所有的接口都被覆盖了，我们就可以开始配置持续集成工具 [Magnum CI](https://magnum-ci.com/) 了。为了使用 Magnum CI，你需要创建一个账号然后设置你的第一个项目。

![](http://ww4.sinaimg.cn/large/7327fe71gw1f4msexy95bj20cy07tmxc.jpg)

接着，给你的项目填入基本信息。

![](http://ww1.sinaimg.cn/large/7327fe71gw1f4msfoh70qj20f409fgm5.jpg)

点击创建你的项目后，就可以开始配置你的项目了。

![](http://ww4.sinaimg.cn/large/7327fe71gw1f4msgxlozjj20qk0e3gn0.jpg)

从这里开始，你可以像例子中一样，选择你用于构建的 git 分支。或者，你可以跳过自定义设置，简单地采用 Magnum CI 的默认值创建你的第一个测试构建配置。

![](http://ww1.sinaimg.cn/large/7327fe71gw1f4mvnqxn8oj20v90az401.jpg)

之后你就会收到一封邮件，上面会详细描述了你的项目构建状态。你还可以增加一个网页钩子，让它将信息推送到你在 [Slack](https://slack.com/) 中的项目房间中。我的团队非常喜欢这个功能，因为这样能够让所有人在忙着工作修改 bug 的时候及时得到通知。如果有人推送了一次代码导致 API 出问题，我们立刻能得到通知并开始着手修复。

总的来说，我的团队都觉得这套测试方法不错，它节省了我们许多时间，并减少了在 QA 过程受挫的次数。

你也想要来一套吗？快来 [clone 我的 git 代码](https://github.com/kathrynhough/node-api-testing-example)，开始亲自运行测试样例吧。


> 翻译自：[MAKE YOUR NODE.JS API BULLETPROOF: HOW TO TEST WITH MOCHA, CHAI, AND SUPERTEST](https://developmentnow.com/2015/02/05/make-your-node-js-api-bulletproof-how-to-test-with-mocha-chai-and-supertest/)