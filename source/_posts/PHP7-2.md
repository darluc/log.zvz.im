title: PHP 7 有些什么值得期待？（二）
date: 2016-1-17 03:13:27
tags: 
- php
---

> 在上一期我们已经介绍了，PHP7中最重要的对于不一致性的修复，以及两个大的新特性。本文将要介绍PHP7的另外6个你会乐于知道的特性。

## Unicode编码的转义语法

新增的转义字符，`\u`，允许我们在php字符串中直接加入Unicode编码字符：

使用语法如下`\u{编码}`，例如绿色的心形 💚，可以直接用PHP字符串表示："\u{1F49A}"。
<!--more-->
## 空值合并操作符

又一个新的操作符，空值合并操作符，`??`。返回左操作数如果其不为**NULL**，否则返回右操作数。

而且很重要的一点是，当左操作数未定义时，不会产生提示错误。此特性更像是`isset()`函数，而不像`?:`三元操作符。

你可以链式使用此操作符，它将返回第一个非空值。

``` php
$config = $config ?? $this->config ?? static::$defaultConfig;
```

## 闭包调用时绑定

PHP 5.4版本时，提供了`Closure->bindTo()`和`Closure::bind()`方法，用于改变闭包中$this的绑定对象和作用域，同时复制一个新的闭包。

PHP 7现在加入了一个更加简单的方法`Closure->call()`，使其在调用时绑定$this和作用域。此方法将绑定的对象作为第一个参数，其后的参数作为闭包参数传入，如下：

``` php
<?php
class HelloWorld{
  private $greeting  = "Hello";
}

$closure = function($whom) { echo $this->greeting . ' ' . $whom; }

$obj = new HelloWorld();
$closure->call($obj, 'World'); // Hello World
```

## 成组使用Use申明

如果你曾有从同一个命名空间引入很多类，你可能会更高兴你的开发工具自动帮你完成了这些工作。对于其他没有使用自动完成工具的人来讲，PHP 7现在提供了成组使用Use申明的方法。它使得你能更快地更清晰引入同一个命名空间下的多个类。

``` php
<?php
// 以前
use Framework\Component\SubComponent\ClassA;
use Framework\Component\SubComponent\ClassB as ClassC;
use Framework\Component\OtherComponent\ClassD;
```

``` php
<?php
// 成组申明
use Framework\Component\{
  SubComponent\ClassA,
  SubComponent\ClassB,
  OtherComponent\ClassD
};
```

它也可以与常量和函数的导入混用。

``` php
<?php
use Framework\Component\{
  SubComponent\ClassA,
  function OtherComponent\someFunction,
  const OtherComponent\SOME_CONSTANT
};
```



## 生成器的改进

### 生成器使用Return语句

生成器加入了两个新的特性。第一个就是[生成器使用Return语句](https://wiki.php.net/rfc/generator-return-expressions)，它允许生成器执行完毕后，返回一个值。

在PHP 7之前，如果你尝试返回任何东西，都会报错。然而，现在你可以调用`$generatro->getReturn()`来取得返回值了。

如果生成器还没有返回，或者已经抛出异常了，那么执行`$generator->getReturn()`也会抛出异常。

如果没有Return语句的生成器执行完毕，则返回null值。

下面有个示例：

``` php
<?php
function gen() {
  yield "Hello";
  yield " ";
  yield "World!";
  
  return "Goodbye Moon!";
}

$gen = gen();
foreach ($gen as $value) {
  echo $value;
}
// 第一次输出 "Hello"，接着是" ", "World!"

echo $gen->getReturn(); // Goodbye Moon!
```

### 生成器代理

第二个特性更加令人激动：[生成器代理](https://wiki.php.net/rfc/generator-delegation)。此特性允许你返回另一个可以迭代的结构，并使其被遍历——例如数组，迭代器或者另一个生成器。

子结构的迭代是由最外层的循环来完成的，单层次的循环而非以递归的方式。

当传递数据或异常给生成器时，就像直接调用子结构一样，也是直接传递给子结构的。

代理语法**yield from \<expression\>**，如下：

``` php
<?php
function hello() {
  yield "Hello";
  yield " ";
  yield "World!";
  
  yield from goodbye();
}

function goodbye() {
  yield "Goodbye";
  yield " ";
  yield "Moon!";
}

$gen = hello();
foreach ($gen as $value) {
  echo $value;
}
```

每一次迭代的输出如下：

1. "Hello"
2. " "
3. "World!"
4. "Goodbye"
5. " "
6. "Moon!"

有一点需要小心：因为子结构自身也可能被迭代产生值，所以完全有可能同一个值被多个迭代所返回——如果这不是你想要的，则需要你自己去避免。

## 引擎异常

在之前的PHP中处理并捕获致命错误几乎是不可能的。但随着新增加的[引擎异常](https://wiki.php.net/rfc/engine_exceptions_for_php7)，许多这些错误将能够转化为异常抛出。

现在，当一个致命错误发生时，将抛出异常，这样你就可以优雅地处理它了。如果你不去处理它，则会产生传统的不可捕获的致命错误。

这些异常是**\EngineException**的实例，且与用户空间的异常不同，它并没有集成基础的**\Exception**类。这样做是为了确保现有代码中捕获**\Exception**的地方不会变得能够捕获到这些致命错误并继续执行下去。这样就保证了代码向后兼容。

以后，如果你想一起捕获传统的异常和引擎异常，那么你需要使用到新增的、它们所共有的父类**\BaseException**。

另外，在`eval()`函数使用时的解析错误会抛出一个**\ParseException**，同时类型不匹配会抛出一个**\TypeException**异常。

这儿有个例子：

``` php
<?php
try {
  nonExistentFunction();
} catch (\EngineException $e) {
  var_dump($e);
}

object(EngineException)#1 (7) {
  ["message":protected] => string(32) "Call to underfined function nonExistantFunction()"
  ["string":"BaseException":private] =>
  string(0) ""
  ["code":protected] =>
  int(1)
  ["file":protected] =>
  string(17) "engine-exceptions.php"
  ["line":protected] =>
  int(1)
  ["trace":"BaseException":private] =>
  NULL
}
```



> 翻译自：[What to Expect When You're Expecting: PHP 7, Part 2](https://blog.engineyard.com/2015/what-to-expect-php-7-2)