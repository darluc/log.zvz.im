title: Javascript 异步探奇：Aysn/Await
date: 2017-12-12 16:23:02
tags:
- javascript
- Nodejs
---

![](https://ws1.sinaimg.cn/large/7327fe71gy1fmawg0jg66j20q80catae.jpg)

### Async/Await 为何物？

Async/Await 即是异步函数（Async Functions），是 JavaScript 中控制异步流程的特殊语法。目前，大多数主流浏览器已经支持了此语法。它的诞生灵感来源于 C# 和 F# 编程语言。目前 Aysnc/Await 已进入 JavaScript/EcmaScript 2017 标准。
<!-- more -->

简单来说，一个异步函数 `async function` 就是一个返回值为 `Promise` 对象的函数 `function`。在 `async function` 异步函数中才可以使用 `await` 关键字。Await 关键字应放在返回值为 Promise 对象的表达式之前，然后它会从这个 Promise 中取得解决值，虽然看上去像是一条同步执行的语句，但实际 Promise 的执行过程仍然是异步的。举个例子往往能解释得更清楚😁。

```javascript
// 这是一个普通的函数，它会返回一个 promise。
// 该 promise 于 2 秒后解决为 "MESSAGE"。
function getMessage() {
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve("MESSAGE"), 2000);
  });
}

async function start() {
  const message = await getMessage();
  return `The message is: ${message}`;
}

start().then(msg => console.log(msg));
// "the message is: MESSAGE"
```

### 为何使用 Async/Await ？

Async/Await 为异步执行的代码提供了一种与同步代码相同的书写方式。它还为异步错误处理提供了一种简洁直接的处理方式，因为它利用了 `try..catch` 的语法结构，这与一般的同步代码处理错误的方式一致。

在进一步深入之前，我们必须强调一个前提：Async/Await 是绝对依赖于 [JavaScript Promises](https://medium.com/@BenDiuguid/asynchronous-adventures-in-javascript-promises-1e0da27a3b4) 的，想要完全理解 Async/Await 就必须先了解它。

## 语法

### Async 函数

创建一个异步函数 `async function` ，只需要在函数申明前加上 `async` 关键字即可，如下：

```javascript
async function fetchWrapper() {
  return fetch('/api/url/');
}

const fetchWrapper = async () => fetch('/api/url/');

const obj = {
  async fetchWrapper() {
    // ...
  }
}
```

### Await 关键字

```javascript
async function updateBlogPost(postId, modifiedPost) {
  const oldPost = await getPost(postId);
  const updatedPost = { ...oldPost, ...modifiedPost };
  const savedPost = await savePost(updatedPost);
  return savedPost;
}
```

在这里 `await` 被用在返回 [promise](https://medium.com/@BenDiuguid/asynchronous-adventures-in-javascript-promises-1e0da27a3b4) 的函数（现在也可称之为异步函数 `aync functions`）前。在函数的第一行 oldPost 被赋予了异步函数 `getPost` 返回的解决值（resolved value）。接下来，我们使用了[对象展开操作符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_operator) 对 `oldPost` 和 `modifiedPost` 进行了一次浅层合并（shallow merge）。最后，我们保存修改过的文章，再一次使用 `await` 关键词接收 `savePost` 函数异步执行返回的 [promise](https://medium.com/@BenDiuguid/asynchronous-adventures-in-javascript-promises-1e0da27a3b4) 对象。

## 例子/常见问题

> ✋ “怎样进行错误处理的？”

好问题！在 async/await 的帮助下，我们可以使用与同步代码一样的语法，`try…catch`。下面的代码中，如果我们的异步调用 `fetch` 返回了某种错误，比如 404，它将会被 `catch` 代码捕获，之后我们就可以对这个错误进行处理了。

```javascript
async function tryToFetch() {
  try {
    const response = await fetch('/api/data', options);
    return response.json();
  } catch(err) {
    console.log(`An error occured: ${err}`);
    // Instead of rethrowing the error
    // Let's return a regular object with no data
    return { data: [] };
  }
}

tryToFetch().then(data => console.log(data));
```

> ✋ “我还是不明白为什么 async/await 语法比 callbacks/promises 语法好。”

问得好！下面有个代码例子可展现出它们之间的区别。假设我们要异步地 **fetchSomeData** 获取某些数据，得到数据后再异步地 **processSomeData** 处理这些数据，如果有错误出现，只简单地返回一个对象。

```javascript
// we have fetchSomeDataCB, and processSomeDataCB
// NOTE: CB stands for callback

function doWork(callback) {
  fetchSomeDataCB((err, fetchedData) => {
    if(err) {
      callback(null, [])
    }
    
    processSomeDataCB(fetchedData, (err2, processedData) => {
      if(err2) {
        callback(null, []);
      }
      
      // return the processedData outside of doWork
      callback(null, processedData);
    });
  });
}

doWork((err, processedData) => console.log(processedData));
```

```javascript
// we have fetchSomeDataP, and processSomeDataP
// NOTE: P means that this function returns a Promise

function doWorkP() {
  return fetchSomeDataP()
    .then(fetchedData => processSomeDataP(fetchedData))
    .catch(err => []);
}

doWorkP().then(processedData => console.log(processedData));
```

```javascript
// we have fetchSomeDataP, and processSomeDataP
// NOTE: P means that this function returns a Promise

async function doWork() {
  try {
    const fetchedData = await fetchSomeDataP();
    return processSomeDataP(fetchedData);
  } catch(err) {
    return [];
  }
}

doWork().then(processedData => console.log(processedData));
```

<center><small>Callback vs Promise vs Async/Await</small></center>

> *✋* “如何处理并发”

如果我们想要异步过程被顺序地处理，只需要简单地使用多行 `await` 语句，并可将一个异步调用的输出结果传递给另一个异步调用，就像平时使用 promises 那样。不过为了理解并发，我们就必须使用 `Promise.all` 方法。如果想让 3 个异步动作同时执行（并发地），我们就要在 `await` 这些 promises 之前，让它们全部开始执行。

```javascript
// Not ideal: This will happen sequentially
async function sequential() {
  const output1 = await task1();
  const output2 = await task2();
  const output3 = await task3();
  return combineEverything(output1, output2, output3);
}
```

上面的代码并不好使，因为 task2 会一直等到 task1 完成之后才开始执行，而 task3 则一直要等到 task1 和 task2 都完成之后才开始，其实这些任务之间并没有依赖关系。理想的做法是，我们让三个任务同时开始执行，进入 `Promise.all` 中等待它们完成。

```javascript
// Ideal: This will happen concurrently
async function parallel() {
  const promises = [
    task1(),
    task2(),
    task3(),
  ];
  const [output1, output2, output3] = await Promise.all(promises);

  return combineEverything(output1, output2, output3);
}
```

在这段代码中，我们同时启动了三个异步任务，并将它们返回的 promises 存入一个数组。然后，将这组已执行的 promises 传入 `Promise.all()`，并 `await` 等待其返回输出。

### 其它说明

* 经常容易忘记的一点：每当你使用 `await` 关键字的时候，你必须在 `async function` 异步函数中使用。
* 当你使用 `await` 时，它只会在 `aync function` 异步函数内产生暂停效果。如下面的例子中，会在其它的日志输出前打印出 `'wanna race?'` 。

```javascript
const timeoutP = async (s) => new Promise((resolve, reject) => {
  setTimeout(() => resolve(s*1000), s*1000)
});

[1, 2, 3].forEach(async function(time) {
  const ms = await timeoutP(time);
  console.log(`This took ${ms} milliseconds`);
});

console.log('wanna race?');
```

当这些 promises 进入等待状态 `awaited`-ed 之后，执行流程会回到主线程， `forEach` 语句外面的 `console.log` 就会被执行。

### 浏览器支持

目前浏览器的支持情况，你可以查看这个[浏览器兼容表](http://caniuse.com/#feat=async-functions)。

### Node 支持

自 `node 7.6.0` 向上都支持 Aysnc/Await 语法！



翻译自：[Asynchronous Adventures in JavaScript: Async/Await](https://medium.com/dailyjs/asynchronous-adventures-in-javascript-async-await-bd2e62f37ffd)