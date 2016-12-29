title: Javascript 中的 UTF-8
date: 2016-12-30 02:50:08
tags: 
- javascript
---


> 译者的话：最近在工作遇到一个小的问题。某段 JS 尝试使用 `decodeURIComponent` 函数对 `escape` 编码过的字符串进行解码导致了浏览器报错：**URIError: URI malformed**。于是上网找寻答案，看到这篇文章。



在 Javascript 中处理编码问题有时会令人烦恼。我最近一次遭遇就是在使用浏览器的 `atob` 和 `btoa` 函数时。这些函数用于二进制字符串和 Base64 ASCII 码之间的转换。但是用它们处理 Unicode 字符串时就挂了：

```shell
> btoa('\u0227');
Error: INVALID_CHARACTER_ERR: DOM
Exception 5
```

[MDN 上关于 `btoa` 的文档](https://developer.mozilla.org/en/DOM/window.btoa#Unicode_Strings)指出下面[来自于 Johan Sundström 的函数](http://ecmanaut.blogspot.com/2006/07/encoding-decoding-utf8-in-javascript.html)可以解决 Unicode 问题：

```javascript
function encode_utf8(s) {
  return unescape(endcodeURIComponent(s));
}

function decode_utf8(s) {
  return decodeURIComponent(escape(s));
}
```

用上这些函数后，`btoa` 就能够给出期望的结果了：

```shell
> btoa(encode_utf8('\u0227'));
"yKc=="
```

Johan 的函数能正常工作，只是让人觉得有点像是黑魔法。所以我决定研究一下 `encode_utf8` 函数的各个部分，搞明白它是如何工作的。
<!-- more -->
## encodeURIComponent

`encode_utf8` 函数中先使用了 `encodeURIComponent` 。`encodeURIComponent` 在[最近的 ECMAScript 规范](http://ecma-international.org/ecma-262/5.1/)中定义如下：

> encodeURIComponent 函数计算产生一个新的 URI，其中的某些字符被替换为一个，二个，三个，或四个字节的 该字符 UTF-8 编码的转义序列。

令我惊讶的是 `encodeURIComponent` 函数是与 UTF-8 编码绑定的。而不像其它语言的类似函数，编码方式是作为函数参数传入的。如果你本来就像使用 UTF-8 编码，当然没有问题，如果你想使用别的编码方式就不好使了（当然也有别的方法[使用 ArrayBuffers 实现定长编码](http://updates.html5rocks.com/2012/06/How-to-convert-ArrayBuffer-to-and-from-String)）。

这儿有个例子是对 Unicode 字符 U+0227（ȧ，小写的拉丁字母 A 顶上加个点）使用 `encodeURIComponent` 函数：

```shell
> encodeURIComponent('\u0227');
"%C8%A7"
```

结果是一个[百分号编码](http://en.wikipedia.org/wiki/Percent-encoding)的字符串，每一段表示该字符串 UTF-8 编码的一个字节。查找 [Unicode 字符 U+0227 的文档](http://www.fileformat.info/info/unicode/char/227/index.htm)，我们可以验证该字符 UTF-8 编码的十六进制形式就是 0xC8 0xA7。

ECMAScript 的定义很模糊，它说 `encodeURIComponent` 会替换“某些字符”，却没说明这些字符。不过一个[快速的试验](http://monsur.hossa.in/temp/encodeURIComponent.html)显示，任何大于 0x7E 的字符都会被编码，所以我觉得说任何非 ASCII 字符都会被 `encocodeURIComponent` 编码肯定不会错。由于百分号编码方式只使用了数字 0-9，字母 A-F 以及 ‘%’ 字符，可以保证输出结果一定是 ASCII 字符串。

现在我们可以先暂停一下。`btoa` 函数所需要的就是 ASCII 字符串，`encodeURIComponent` 函数就已然够用了：

```shell
> btoa(encodeURIComponent('\u0227'));
"JUM4JUE3"
```

不过这样做有一些缺陷。首先，输出的结果字符串与输入的字符串的二进制变了。这只是个原始字节的一个编码版本。这可能在与其它系统交互时造成困难和烦恼。

第二个缺陷是，`encodeURIComponent` 函数生成了一个更长的字符串。单个 Unicode 字符采用 UTF-8 编码最多占用 4 个字节，经过 URI 编码后就能产生一个 12 个字符长的串。之后在经过 Base64 编码（能使字符串变得更长），最终输出的结果比输入的字符能长许多倍。为了解决字符串长度问题，Johan 找来了 `unescape` 函数。

## unescape

 与 `encodeURIComponent` 函数不同，`escape` / `unescape` 函数在最近的 ECMAScript 版本中都没有被定义。[JavaScript: The Definitive Guide](http://shop.oreilly.com/product/9780596805531.do) （第6版，855页）表述如下：

> 尽管在第一版的 ECMAScript 中 unescape 被标准化了，但自从 ECMAScript v3 起它就被弃用并从标准中移除了。而 ECMAScript 的实现中，这个方法很大可能上都被实现了，不过这不是必须的。

鉴于这两个函数从浏览器诞生之初就存在了，我认为它们不会很快就消失不见。

`unescape` 函数用于将百分号形式编码的字符串反转义。它可以用于标准的百分号转义形式（%XX），也可以用于非标准的 Unicode 百分号形式（%uXXXX）。以下是最初那个例子的反转义结果：

```shell
> var e = encodeURIComponent('\u0227');
console.log(e);
"%C8%A7"
> var u = unescape(e); console.log(u);
"È§"
```

“È§”是 C8 和 A7 所对应的 ASCII 字符，如下可见：

```shell
> u.charCodeAt(0).toString(16);
"c8"
> u.charCodeAt(1).toString(16);
"a7"
```

调用和 `unescape` 与调用 `decodeURIComponent` 非常不同。`decodeURIComponent` 将字符串转换为 UTF-8 编码，它会将两个被百分号编码的字符合起来解译为初始字符的 UTF-8 表现形式。`unescape` 则只是将字符返回不做任何解译。

使用 `unescape` 函数的好处是，我们可以使用更段的字符串保留实际的 UTF-8 字节：

```shell
> e.length
6
> u.lenght
2
```

新的字符串中的字节是原始字符串的 UTF-8 编码字节作为 ASCII 码解译时表现出来的样子。

## 总结

综上所述， `encode_utf8` 函数对字符串转换的整个过程就显而易见了。

Johan 的小小黑魔法能够工作，是因为它将 Unicode 字符串转为编码过 UTF-8 字符串，然后再将编码字节对应转换为对应的 ASCII 字符串并返回。我喜欢这个方法是因为它利用了浏览器有多种编码函数的事实。如果只是分别成对使用 `encodeURIComponent` / `decodeURIComponent` 或者 `escape` / `unescape` 函数，则相当于空操作。Johan 的天才之处就在于他同时使用了两种不同的编码函数，从而得到了我们想要的结果。



> 译者的话：文中有一句话可以解释我所遇到的问题，就是 **The `unescape` function unescapes a percent-encoded string. It works with the standard percent-encoding (%XX) as well as the non-standard Unicode (%uXXXX) percent encoding.** `unescape` 函数能够支持非标准的百分号编码方式，而 `decodeURIComponent` 函数却不能，当你所操作的字符串在 ASCII 范围内时，两者其实是可以互换使用的。有兴趣可以自己尝试一下。
>
> 
>
> 原文地址：[UTF-8 in Javascript](http://monsur.hossa.in/2012/07/20/utf-8-in-javascript.html)