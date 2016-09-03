title: 搞不明白的 Javascript 闭包
date: 2016-09-03 23:32:18
tags: 
- javascript
---

在 Javascript 语言中，闭包就是一个函数，只是它上下文中的变量以引用的形式与其绑定在了一起。

```javascript
function getMeAClosure() {
  var canYouSeeMe = "here I am";
  return (function theClosure() {
    return {canYouSeeIt: canYouSeeMe ? "yes" : "no"};
  });
}

var closure = getMeAClosure();
closure().canYouSeeIt; //"yes"
```

实际上每个 Javascript 函数在生成的时候都形成了闭包。稍后我会给大家解释闭包的产生原因和过程，然后纠正一些关于闭包的错误概念，最后再给出一些闭包的实际用例。不过首先简单介绍一下闭包相关的基础概念：Javascript 的闭包是通过 `词法域 (lexical scope)` 和 `变量环境 (VariableEnvironment)` 实现的。
<!--more-->

## 词法域（ Lexical Scope ）

“词法”这个词一般都是语言相关。所以函数的词法域是静态的，是由函数代码在源代码中的位置决定的。

参考以下代码：

```javascript
var x = "global";
function outer() {
  var y = "outer";
  function inner() {
    var x = "inner";
  }
}
```

函数 `inner` 在代码中被函数 `outer` 包裹着，而 `outer` 又被全局上下文包含在内。这样就形成了一个词法继承关系：

global

— outer

—— inner

每个函数的外部词法域都是由词法继承关系中它的祖先决定的。因此，`inner` 函数的外部词法域就是由全局对象和函数 `outer` 组成的。

## 变量环境（ VariableEnvironment ）

全局对象有一个相关的执行上下文。而且每一次函数调用也会建立并进入一个新的执行上下文。这个执行上下文相对于静态的词法域是动态生成的。每一个执行上下文都确定了一个变量环境，它是在该上下文中所声明变量的容器。（[ES 5](http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-262.pdf) 10.4，10.5）

> 注意，在 EcmaScript 3 中，函数的变量环境（ VariableEnvironment ）被称为活动对象（ ActivationObject ）

以下伪代码可以用来描述变量环境

```javascript
//variableEnvironment: {x: undefined, etc.};
var x = "global"
//variableEnvironment: {x: "global", etc.};

function outer() {
  //variableEnvironment: {y: undefined};
  var y = "outer";
  //variableEnvironment: {y: "outer"};
  
  function inner() {
    //variableEnviroment: {x: undefined};
    var x = "inner";
    //variableEnvironment: {x: "inner"};
  }
}
```

不过，这只描述了整个图景中的一部分。每个变量环境都会继承它所属词法域的变量环境。

## [[scope]]属性

当一个函数定义过程发生在某个执行上下文环境中时，会生成一个新的函数对象，此函数对象会包含一个名为 [[scope]] 的内部属性引用当前的变量环境。（[ES 5](http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-262.pdf) 13.0-2）

每个函数都有这样一个 [[scope]] 属性，而且当函数被调用时，这个 [[scope]] 属性会被赋值给变量环境的 *outerLex* 属性，该属性是对外层词法环境的引用（ outer lexical environment reference 简写为 outerLex ）。这样一来，每个变量环境都继承了它父级的变量环境。这个 [[scope]] 链会一直延伸至全局对象，与词法继承的长度一样。

现在让我们再来看一下伪代码：

```javascript
//VariableEnvironment: {x: undefined, etc.};
var x = "global";
//VariableEnvironment: {x: "global", etc.};

function outer() {
  //VariableEnvironment:{y: undefined, outerLex: {x: "global", etc.}};
  var y = "outer";
  //VariableEnvironment: {y: "outer", outerLex: {x: "global", etc.}};
  
  function inner() {
    //VariableEnvironment: {x: undefined, outerLex: {y: "outer", outerLex: {x:"global", etc.}};
    var x = "inner";
    //VariableEnvironment: {x: "inner", outerLex: {y: "outer", outerLex: {x:"global", etc.}};
  }
}
```

对于层层嵌套的变量环境来说，[[scope]] 属性起到了桥接的作用，使得外层变量被嵌入到内层的变量环境中（并且访问优先级与词法接近程度一致）。[[scope]] 属性还成全了闭包，因为假如没有它的话，当外层函数返回时，函数中的变量就会被解除引用而被当作垃圾回收。

这儿就是所谓的闭包出现的地方，它只不过是词法域作用（lexical scoping）时产生的副作用而已😉

## 辟谣

既然知道了闭包是如何工作的，我们就可以开始纠正一些关于闭包的错误理解了。

### 错误1. 闭包是在一个内层函数作为结果返回后才生成的 

当一个函数被创建时，就被赋予了 [[scope]] 属性，该属性引用了外层语法域中的变量并防止它们被回收。所以闭包是在函数创建时就生成的。

并不是说一个函数必须在它被返回之后才成为闭包。以下就是一个函数没有被返回却是一个闭包的例子：

```javascript
var callLater = function(fn, args, context) {
  setTimetout(function(){fn.apply(context, args)}, 2000);
}

callLater(alert, ['hello']);
```

### 错误2. 外层变量的值会被复制或固化在闭包中 

如下例所见，闭包会引用变量而不是变量的值。

```javascript
//Bad Example
//Create an array of functions that add 1,2 and 3 respectively
var createAdders = function() {
  var fns = [];
  for (var i = 0; i < 4; i ++) {
    fns[i] = (function(n) {
      return i + n;
    });
  }
  return fns;
}

var adders = createAdders();
adders[1](7); //11 ??
adders[2](7); //11 ??
adders[3](7); //11 ??
```

这三个加数函数都引用了同一个变量 *i* 。等到三个函数被调用的时候，*i* 的值已经是 4 了。

有一种解决方案是通过调用匿名函数传递每个参数。由于每次函数调用都发生在一个独立的执行上下文中，即使连续调用，我们也可以保证参数变量的唯一性。

```javascript
//Good Example
//Create an array of functions add 1,2 and 3 respectively
var createAdders = function() {
  var fns = [];
  for (var i = 1; i < 4; i ++) {
    (function(i){
      fns[i] = (function(n) {
        return i + n;
      });
    })(i)
  }
  return fns;
}

var adders = createAdders();
adders[1](7); //8 :-)
adders[2](7); //9 :-)
adders[3](7); //10 :-)
```

### 错误3. 闭包一定是内层函数

不可否认外层函数所创建的闭包毫无意思，因为它的 [[scope]] 属性只是引用了全局变量域而已，在任何情况下都是可以访问到的。对于每个函数来说闭包的产生过程都是一样的，而且每个函数都会产生闭包，记住这点很重要。

### 错误4. 闭包一定是匿名函数

我在许多文章里看到过这种说法。悲哀！实在是悲哀！😉

### 错误5. 闭包会导致内存泄漏

闭包本身并不会产生循环引用。在文章开头的例子中，`inner` 函数通过它的 [[scope]] 属性引用了外层变量，但是外层变量和 `outer` 函数都没有引用 `inner` 函数或其内部变量。

老版本的 IE 浏览器因内存泄漏而声名狼藉，闭包却替它背了锅。一个典型的起因是，一个 DOM 元素被函数引用，同时这个 DOM 元素的一个属性引用了此函数词法域中的另一个对象。从 IE6 发展到 IE8 这类循环引用问题绝大多数都已经被解决了。

## 经典用例

### 函数模版

有时我们想要定义一个函数的多个版本，每个版本都遵从同一个蓝图，却能根据提供的参数产生不同的变化。比如，我们可以创建一套标准的度量单位转换函数：

```javascript
function makeConverter(toUnit, factor, offset) {
  offset = offset || 0;
  return function(input) {
    return [((offset + input) * factor).toFixed(2), toUnit].join(" ");
  }
}

var milesToKm = makeConverter('km', 1.60936);
var poundsToKg = makeConverter('kg', 0.45460);
var farenheitToCelsius = makeConverter('degree C', 0.5556, -32);

milesToKm(10); //"16.09 km"
poundsToKg(2.5); //"1.14 kg"
farenheitToCelsius(98); //"36.67 degrees C"
```

如果和我一样，你喜欢函数式抽象，那么再深入一步就是进行函数参数的增量绑定（[currify](https://javascriptweblog.wordpress.com/2010/04/05/curry-cooking-up-tastier-functions/)）了。

### 函数式 Javascript（Functional Javascript）

除去函数——这个Javascript 中第一级别的对象外，函数式 Javascript 最好的伙伴就是闭包了。

bind, [curry](https://javascriptweblog.wordpress.com/2010/04/05/curry-cooking-up-tastier-functions/), [partial](https://javascriptweblog.wordpress.com/2010/05/17/partial-currys-flashy-cousin/) 和 [compose](https://javascriptweblog.wordpress.com/2010/04/14/compose-functions-as-building-blocks/) 的典型实现方式，都依赖于闭包为新函数提供原始函数及参数的引用。

如下例的 curry （参数增量绑定）：

```javascript
Function.prototype.curry = function() {
  if (arguments.length < 1) {
    return this; //nothing to curry with - return function
  }
  var __method = this;
  var args = toArray(arguments);
  return function() {
    return __method.apply(this, args.concat([].slice.apply(null, arguments)));
  }
}
```

我们用 curry 的方式重写上面的例子：

```javascript
function converter(toUnit, factor, offset, input) {
  offset = offset || 0;
  return [((offset + input) * factor).toFixed(2), toUnit].join(" ");
}

var milesToKm = converter.curry('km', 1.60936, undefined);
var poundsToKg = converter.curry('kg', 0.45460, undefined);
var farenheitToCelsius = converter.curry('degrees C', 0.5556, -32);

milesToKm(10); //"16.09 km"
poundsToKg(2.5); //"1.14 kg"
farenheitToCelsius(98); //"36.67 degrees C"
```

还有许多很漂亮的函数调节器（modifier）也是利用闭包实现的。下面这个如宝石般漂亮的代码，来自于[Oliver Steele](http://osteele.com/) ：

```javascript
/**
 * Returns a function that takes an object, and returns the value of its 'name' property
 */
var pluck = function(name) {
  return function(object) {
    return object[name];
  }
}
 
var getLength = pluck('length');
getLength("SF Giants are going to the World Series!"); //40
```

### 模块化模式

这是个[广为人知的技术](https://javascriptweblog.wordpress.com/2010/04/22/the-module-pattern-in-a-nutshell/)，它使用闭包维护了一个私有的、独占的对外层域中变量的引用。这里我用模块化模式实现了一个“猜数字”游戏。注意在此例中，闭包（*guess*）与 *secretNumber* 变量有着唯一的联系，而 *responses* 对象则只在创建时使用了它的值。

```javascript
var secretNumberGame = function() {
  var secretNumber = 21;
  
  return {
    responses:{
      true: "You are correct! Answer is " + secretNumber,
      lower: "Too high!",
      higher: "Too low!"
    },
    
    guess: function(guess) {
      var key = (guess == secretNumber) || (guess < secretNumber ? "higher" : "lower");
      alert(this.responses[key]);
    }
  }
}

var game = secretNumberGame();
game.guess(45); //"Too high!"
game.guess(18); //"Too low!"
game.guess(21); //"You are correct! Answer is 21"
```

## 总结

在编程术语中，闭包代表了优雅和精致。它使得代码更简洁漂亮、可读性更高，同时提升了代码的重用性。了解闭包的工作原理，就可以避免使用中的不确定性。我希望这篇文章能够帮到你，如果有问题、想法或其它事，都请留下您的评论。

## 进一步的阅读

[ECMA-262 5th Edition](http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-262.pdf)

* 10.4 创建变量环境
* 10.4.3.5-7 在变量环境中引用 [[scope]] 属性
* 10.5 填入变量环境
* 13.0-2 函数创建时为 [[scope]] 属性赋值





> 翻译自：[Understanding Javascript Closures](https://javascriptweblog.wordpress.com/2010/10/25/understanding-javascript-closures/)