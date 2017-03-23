title: 为什么柯里化有用
date: 2017-3-23 15:11:23
tags:
- javascript
---

写一段可以无限被重用的代码，对于程序员来说无异于黄粱美梦。代码表达清晰是因为代码要表达需求，代码易于重用……好吧，只是因为你要重复使用它。两者不可得兼，你还能奢求更多么？

[柯里化](https://npmjs.org/package/curry)可以帮上忙。

## 什么是柯里化，为何它如此美味？

通常， JavaScript 函数看起来都像这样：

```javascript
var add = function(a, b){ return a + b }
add(1, 2) //= 3
```

它接受一些参数，然后有一个返回值。我可以使用过多（多余的参数会被忽略）或过少（会给出奇怪的返回值）的参数调用它：

```javascript
add(1, 2, 'IGNORE ME') //= 3
add(1) //= NaN
```

柯里化可以使一个多参数函数转化为一系列单参数函数。比如，柯里化后的加法函数：
<!--more-->

```javascript
var curry = require('curry');
var add = curry(function(a, b){ return a + b })
var add100 = add(100)
add100(1) //= 101
```

柯里化后的多参数函数可以如下调用：

```javascript
var sum3 = curry(function(a, b, c){ return a + b + c })
sum3(1)(2)(3) //= 6
```

这样的写法在 JavaScript 中可能有点丑，所以[柯里化](https://npmjs.org/package/curry)也允许你一次传入都个参数：

```javascript
var sum3 = curry(function(a, b, c){ return a + b + c })
sum3(1, 2, 3) //= 6
sum3(1)(2, 3) //= 6
sum3(1, 2)(3) //= 6
```

## 这样又如何？

如果你对于那些经常使用柯里化函数的语言很熟（比如[Haskell](http://learnyouahaskell.com/)），可能看不出来这样做会带来什么好处。在我的理解中，主要有以下两点好处：

* 小片断的函数可以被配置，并很容易得到重用，且代码整洁；
* 代码中彻头彻尾只用函数。

### 小片断函数

举一个比较明显的例子；从一个集合中获取所有成员的 id：

```javascript
var objects = [{ id: 1 }, { id: 2 }, { id: 3 }]
objects.map(function(o){ return o.id })
```

如果你正在厘清第二行代码的真实逻辑，让我帮你把它择出来吧：

> MAP over OBJECTS to get IDS（遍历所有对象获取它们的 ID 值）

从函数定义的形式来看，只是这一行就有很多垃圾代码。让我们将其整理干净：

```javascript
var get = curry(function(property, object){ return object[property] })
objects.map(get('id')) //= [1, 2, 3]
```

现在我们可以照着代码讲出真实的逻辑了 - 遍历所有对象获取它们的 ID 值。我们创建的*get* 方法是一个**可部分配置的方法**。

如果我们想要重用‘从一组对象中获取 id ’这个功能怎么办？最直接的办法就是：

```javascript
var getIDs = function(objects){
    return objects.map(get('id'))
}
getIDs(objects) //= [1, 2, 3]
```

嗯，看起来我们又丢掉了优雅简练，退回到杂乱无章的编码风格了。那我们该怎么办呢？额 - 如果 map 方法可以在还没有数据的时候，用一个函数进行部分配置的话？

```javascript
var map = curry(function(fn, value){ return value.map(fn) })
var getIDs = map(get('id'))

getIDs(objects) //= [1, 2, 3]
```

看起来，如果我们以柯里化函数作为基础构建模块，我们就可以使用它们很容易得创造出全新的功能函数。更让人兴奋的是，代码看起来更能体现你的业务逻辑。

### 连成串的函数

使用这种方式写代码还有另一个好处，它鼓励函数的使用；而不是类的方法。虽然类的方法是很美好的 —— 允许多态，代码可读性高 —— 但它们并不总适合所有的工作，比如拥有大量异步调用的时候。

下面的例子中，我们会从服务器端获取数据，再将其进行处理。数据形式如下：

```javascript
{
    "user": "hughfdjackson",
    "posts": [
        { "title": "why curry?", "contents": "..." },
        { "title": "prototypes: the short(est possible) story", "contents": "..." }
    ]
}
```

你的任务是得到该用户所有文章的标题。现在开始！

```javascript
fetchFromServer()
    .then(JSON.parse)
    .then(function(data){ return data.posts })
    .then(function(posts){
        return posts.map(function(post){ return post.title })
    })
```

好吧，这不公平，是我催得太急了。（我还帮你写了上面这些代码 —— 你也许能更加优雅地解决这个问题，可能我已经跑题了）

由于承诺链（ chains of promises ）（也许你更喜欢称其为回调函数）基本都是与函数一起使用的，你无法简单直接地遍历数据，直到它先从服务器返回并被（无论视觉上或头脑中的）一团乱麻包裹住。

让我们再次出发，这回我们使用已经定义过的方法：

```javascript
fetchFromServer()
    .then(JSON.parse)
    .then(get('posts'))
    .then(map(get('title')))
```

Ok，很少的逻辑，轻松地表达；如果没有柯里化函数我们是无法如此容易地做到的。

## 简而言之

[柯里化](https://npmjs.org/package/curry)能给予你一种令人垂涎的表达能力。

我建议你立刻开始使用它。如果你已经熟稔于此，那么你一定会发现它的 API 接口直接好用。如果还没有，那么你应当与你的同事一起好好考虑一下了。