title: PHP 7 有些什么值得期待？（一）
date: 2015-10-24 23:29:47
tags: 
- php
---
# PHP 7 有些什么值得期待？（一）

大多数人可能已经知道，PHP7是PHP的下一个重要发布版本。

无论你有什么想法，PHP7的发布都将是一个大事件，而且今年就会发布。但是这对你来说意味着什么呢？我们可以发现许多运营站点都不太愿意更新到较新的PHP 5.x版本。本次大版本的发布是否会带来许多向后兼容的问题，而使得这些站点更加不愿意进行更新呢？

答案是：这是因具体情况而异的。且看下文分析。

<!--more-->

在PHP 7中，许多语言边界问题（language edge cases）都得到了清理。另外，性能和不一致性的修复在此版本都是关注的焦点。

让我们看看详细情况。

## 对于不一致性的修复

遗憾的是，needle/haystack问题没有得到修复。不过，两个重要的RFC获得通过，它们能带来更加重要的内部和用户空间一致性。

最大（且最难以察觉）的变化是[Abstract Syntax Tree(AST)抽象语法树)](https://wiki.php.net/rfc/abstract_syntax_tree)的加入——代码编译过程的中间表示法。它可以使我们能够清除一些边界情况下的不一致问题，同时为未来一些惊人的工具开启了无限可能，例如使用AST产生更高性能opcodes。

第二点是引入了[Uniform Variable Syntax 统一变量语法](https://wiki.php.net/rfc/uniform_variable_syntax)，可能会给你带来更多的问题。它解决了表达式求值上严重的不一致问题。比如，可以使用 `($object->closureProperty)()`调用闭包类型属性，而且可以链式调用静态方法，如下：

``` php
class foo { static $bar = 'baz';}
class baz { static $bat = 'Hello World';}

baz::$bat = function () { echo "Hello World"; };

$foo = 'foo',
($foo::$bar::$bat)();
```

不过，有些语义也受影响产生了变化。特别是变量值作为变量和变量值作为属性时。

或者说变得更简洁了，如下的语句：

``` php
$obj->$properties['name']
```

在PHP5.6中，会被解释为：

``` php
$obj->{$properties['name']}
```

而在PHP7中，则被解释为：

``` php
{$obj->$properties}['name']
```

使用变量值变量只是一般的边界情况，更加令我难以接受地是，变量值属性的变化。还好我们可以采用加大括号的方式，让这些情况在PHP5.6和7下表现一致。

## 性能提升

升级至PHP 7的最大动力来自于它的性能提升，这主要得益于被称作[phpng](https://wiki.php.net/rfc/phpng)的修改。性能的提升或许可以保证更快被采用，特别是那些平时不太乐意升级的较小的主机供应商，因为通过升级，他们能够为更多的客户提供服务，而不需要对硬件进行升级。

取决与参照哪家的benchmark程序，PHP 7的性能已经能与Facebook的HHVM一较高下了。HHVM使用了JIT(Just In Time)编译器将PHP代码直接编译为机器指令（在可能的情况下）。

尽管有许多相关的讨论，PHP 7依然没有使用JIT。暂时还不清楚在使用JIT的情况下能否获得更多的性能提升，但是如果有人做了一个，那一定会很有意思。

除了性能提升外，PHP 7 还大量提升了内存使用效率。对于内部数据结构的优化，不仅节省了内存空间，也是提升性能的主要手段之一。

## 无法完全向后兼容

尽管内核开发者们尽力保证向后兼容，可惜当语言进化发展的时候，并不是总能如愿的。

不过，和前面提到的统一变量语法带来的对向后兼容性的破坏一样，绝大多数破坏都是微不足道的，例如下面的**对非对象类型变量进行方法调用时产生的可捕获致命错误**：

``` php
set_error_handler(function($code, $message) {
  var_dump($code, $message);
});

$val = null;
$var->method();
echo $e->getMessage(); // Fatal Error: Call to a member function method() on null
echo "Hello World"; // Still runs
```

此外ASP和脚本标签都被移除了，意味着<%和<%=，或者&lt;script language=“php”&gt;（以及它们对应的闭合标签，%>，和&lt;/script&gt;）都不能再被使用。

其它较大的变化可以参考[移除所有的过时函数](https://wiki.php.net/rfc/remove_deprecated_functionality_in_php7)。

最重要的是POSIX兼容的正则表达式扩展的移除，ext/ereg（5.3时被弃用）和旧的ext/mysql扩展（5.5时被弃用）。

另一个较小的向后不兼容变化是不再允许一条switch语句中包含多个default子句。在PHP 7以前，以下的语句是合法的：

``` php
switch($expr) {
  default:
  	echo "Hello World";
    break;
  default:
  	echo "Goodbye Moon!";
    break;
}
```

最后的一条default子句将被执行。而在PHP 7中，则会产生如下结果：

``` php
Fatal error: Switch statements may only contain one default clause
```

## 新的特性

我们小心地处理向后不兼容性，我们赞赏性能的提升，我们陶醉在新的特性中！新特性才是使每个发行版本中有意思的部分，PHP 7可有不少新特性哦。

## 标量类型提示以及返回类型

让我们从PHP 7最具争议的新特性开始讲起：[标量类型提示](https://wiki.php.net/rfc/scalar_type_hints_v5)。加入此特性的相关投票才刚刚通过，投票发起人就永久退出了PHP的开发（同时撤销了RFC）。随之而来的，多个RFC文档提出此特性的实现方案，同时产生了许多公开的辩论，最终在初始的RFC得到通过之后，一切才归于平静。

对于我们来说，作为最终使用者，对于标量类型的类型提示到底意味着什么呢。具体来说就是int，float，string和bool类型。类型提示默认是不严格的，它们会将原始类型数据强制转换为提示所指定的类型。所以，当你将int(1)作为参数传递给一个需要float类型参数的函数时，它会被转换为float(1.0)。将float(1.5)传递给需要int类型的函数时，则会被转换为int(1)。

下面举例说明：

``` php
<?php
function sendHttpStatus(int $statusCode, string $message) {
  header('HTTP/1.0 ' .$statusCode. ' ' .$message);
}

sendHttpStatus(404, "File Not Found"); // integer and string passed
sendHttpStatus("403", "OK"); // string "403" coerced to int(403)
```

而且，你可以开启严格模式：在文件顶部加入代码行`declare(strict_types=1);`，就可以使得在当前文件中的所有函数调用都采用严格的类型检查模式。严格模式是由调用函数的文件决定的，而不是声明函数的文件决定的。

如果函数类型提示检测到不匹配的情况，就会产生一个可捕获的致命错误：

``` php
<?php
declare(strict_types=1); // must be the first line

sendHttpStatus(404, "File Not Found"); // integer and string passed
sendHttpStatus("403", "OK");

// Catchable fatal error: Argument 1 passed to sendHttpStatus() must be of the type integer, string given
```

此外，PHP 7 还支持返回值类型提示，与参数的类型提示支持相同的数据类型。与[hack](https://blog.engineyard.com/2014/hhvm-hack-php)的语法一样，只需在函数名结尾括号后加上冒号和返回类型：

``` php
<?php
function isValidStatusCode(int $statusCode): bool {
  return isset($this->statuses[$statusCode]);
}
```

上面的例子中，**bool**指明函数返回值为布尔类型。

返回值类型提示的严格模式与参数类型提示遵守同样的规则。

## 组合比较操作符

我个人最喜欢的 PHP 7 新特性是组合比较操作符，**<=>**，也被称作宇宙飞船操作符。我可能有点偏心，因为我写了最初的补丁，并使得（T_SPACESHIP）这个名字流行起来，无论如何这个操作符对PHP语言都是个很好的扩充，带来了大于小于符号所不具备的功能。

它的功能如同**strcmp()**或者**version_compare()**，当左操作数小于右操作数时返回-1，相等时返回0，如果左操作数大于右边则返回1。主要不同在于它可以接受任意类型的操作数，不仅仅只是字符串类型，还包括整形，浮点型，数组等等。

最通常的用法是用于排序回调函数的实现：

``` php
<?php
// Pre Spacefaring^W PHP 7
function order_func($a, $b) {
  return ($a < $b) ? -1 : (($a > $b) ? 1 : 0);
}

// Post PHP 7
function order_func($a, $b) {
  return $a <=> $b;
}
```



**本文翻译自：[https://blog.engineyard.com/2015/what-to-expect-php-7](https://blog.engineyard.com/2015/what-to-expect-php-7)**