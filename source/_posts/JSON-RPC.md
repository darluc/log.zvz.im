title: JSON-RPC 2.0 Specification
date: 2015-10-06 22:49:36
tags:
- json
---
# JSON-RPC

JSON-PRC 是与 XML-RPC类似的轻量级远程过程调用协议。


## JSON-RPC 2.0 说明文档

## 1.概述
JSON-RPC 是一种无状态、轻量级远程过程调用（RPC）协议。本文档主要定义了调用过程中的一些数据结构和规则。该协议是与传输方式无关的：其定义内容在同一个处理过程使用时，可通过sockets、HTTP或其它任意消息传递环境。JSON-RPC使用了[JSON(RFC 4627)](http://www.ietf.org/rfc/rfc4627.txt)作为数据格式。
它是从简设计的！
<!--more-->

## 2.约定
”MUST”，”MUST NOT”，”REQUIRED”，”SHALL”，”SHALL NOT”，”SHOULD"，”SHOULD NOT”，”RECOMMENDED"，”MAY”，和"OPTIONAL"这些关键词由[RFC 2119](http://www.ietf.org/rfc/rfc2119.txt)定义。
JSON-RPC使用了JSON格式，所以与JSON使用相同的类型系统（参见[http://www.json.org/](http://www.json.org/)或者[RFC 4627](http://www.ietf.org/rfc/rfc4627.txt)）。JSON可以表示四种基础类型（字符串，数字，布尔，和空值）以及两种结构类型（对象和数组）。术语”Primitive”在本文中指代四种基础类型，术语”Structured”表示两种结构类型。文中的JSON类型首字母总是大写的，例如：**Object**，**Array**，**String**，**Number**，**Boolean**，**Null**。**True**和**False**同样采用首字母大写的方式。
客户端和服务端之间交互的所有成员名称都应当是大小写敏感的。术语：**函数**，**方法**和**过程**可以互换使用。
客户端被定义为请求对象的来源和应答对象的处理者；服务端被定义为应答对象的来源和请求对象的处理者。
本文档的实现很容易充斥着这两种角色，同时针对不同的客户端或者同一个客户端。在这里并不讨论其复杂性层面的问题。

## 3.兼容性
JSON-RPC 2.0 请求对象和应答对象可能与现存的JSON-RPC 1.0客户端端或者服务端无法兼容。但是它们还是很容易区分的，因为2.0版本总是包含一个名为”jsonrpc”成员，其值为”2.0”，而1.0版本则没有。绝大部分时候2.0的实现应当尝试处理1.0版本的对象，即使不是1.0的点对点和类提示方面。

## 4.请求对象`Request Object`
一次RPC调用表现为向服务器发送的一个请求对象。该对象包含以下成员变量：

**jsonrpc**
  指示JSON-PRC版本的字符串。必须等于”2.0"

**method**
  字符串类型，包含被调用方法的名称。方法名称由rpc开头，紧随着一个.字符( U+002E 或者 ASCII 46)，是rpc内部保留的方法和扩展名称，不允许被使用。

**params**
  结构类型值，保存着方法调用时的参数值。此成员可省略。

**id**
  由客户端指定的标识，必须包含一个字符串或数字或NULL值。如果对象中不包含此成员，则改对象被认作一个通知(notification)。id值通常不应该为NULL[^1]，而且数字值不应当包含小数部分[^2]。

如果包含id值，则服务端的响应必须包含与其一样的值。此成员用于维护两个对象之间的关联关系。

[^1]: 不鼓励使用Null作为id的值，因为规范文档使用Null值作为id未知时响应对象id的值，而且，1.0版本中使用id为null值的对象被视作通知，这样做容易在处理时造成混乱。​
[^2]: 小数部分可能是有问题的，因为许多十进制小数都无法用二进制准确的表示。

### 4.1通知 `Notification`
通知是一个不包含”id”的请求对象。一次通知请求意味着客户端不需要相应的应答对象，因此不需要有应答对象返回给客户端。服务端必须不能回应通知，包括哪些在批量请求中的消息。
通知无法通过定义确认，因为它们没有对应返回的应答对象。而且，客户端无法得知任何错误消息（比如：”Invalid params”, “Internal error”)。

### 4.2 参数结构
RPC调用包含的参数必须是结构类型的。要么使用数组，按位置区分；要么使用对象，按名称区分。
- 按位置区分：params必须是一个数组，以服务端所期望的顺序包含所有参数；
- 按名称区分：params必须是一个对象，成员名称与服务端期望的参数名称一直。如果缺少相应名称的参数，则可能产生错误。而且名称必须与方法所期望的参数名称完全匹配，包括大小写。

## 5 应答对象`Response Object`
当发生RPC调用时，除非是通知消息，否则服务端**必须**响应一个应答对象。应答对象表现为单个JSON对象，包含以下成员对象：

**jsonrpc**
  指示JSON-PRC版本的字符串。必须等于”2.0"

**result**
  如果调用成功则必须包含此成员。
  如果发生错误则必须不包含此成员。
  其值由被调用的服务端方法决定。

**error**
  此成员在发生错误时是必要的。
  如果在调用时没有发生错误，则此成员必须不存在。
  此成员值必须是一个对象，在5.1节中有具体描述。

**id**
  此成员是必要的。
  它必须与对应请求对象的id值相同。
  如果在获取请求对象的id时发生错误，那么它必须为null值。

result成员和error成员两者必须包含其一，且不能同时存在。

### 5.1 错误对象`Error Object`
当RPC调用发生错误时，应答对象必须包含一个错误成员对象，该对象包含以下成员：

**code**
  用于指示发生错误的数字。
  必须是整数。

**message**
  用于简短描述该错误的字符串。
  该消息内容应当是一个简洁明了的单句。

**data**
  基础类型或者结构类型值，包含更多关于该错误的信息。
  此成员是可以省略。
  成员值是由服务端定义的（比如详细错误信息，嵌套的更多错误等等）。

从_-32768_至_-32000_的错误代码是保留的预定义错误代码。在此范围中，但未出现在下方列表中的代码，是预留给将来使用的。这些错误代码基本上与XML-RPC的相同，参考链接地址[http://xmlrpc-epi.sourceforge.net/specs/rfc.fault_codes.php]()。

| **code**        | **message**      | **meaning**                      | 
| --------------- | ---------------- | -------------------------------- | 
| -32700          | Parse error      | 服务端接收到无效的JSON数据。在解析JSON字符串时发生错误。 | 
| -32600          | Invalid Request  | 发送出的JSON不是一个有效的请求对象。             | 
| -32601          | Method Not found | 方法不存在或者不可用。                      | 
| -32602          | Invalid params   | 无效的方法参数。                         | 
| -32603          | Internal error   | JSON-RPC内部错误。                    | 
| -32000 ~ -32099 | Server error     | 由具体实现定义的服务端错误。                   | 

剩余错误代码空间，可以由应用自定义错误。

## 6 批量操作
客户端可以通过发送请求对象数组来发起一次批量请求。
服务端则应该在所有调用请求完成后，返回一个包含对应应答对象的数组。每个请求对象都应该有对应的应答对象，通知则不应该有应答对象。服务端可以以并发的方式处理批量RPC调用，而不必考虑并行顺序和处理时长。
返回数据中的应答对象可以以任意顺序排列。客户端应当使用id成员来关联对应的请求对象和应答对象。
如果批量RPC调用本身没有被正确识别为JSON或者至少包含一个值的数组，那么服务端的响应必须是单独一个应答对象。如果在应答数组中没有应答对象，服务端必须不能返回一个空数据，而是不返回任何数据。

## 7 举例
语法：

``` javascript
--> 发送给服务端的数据
<-- 发送给客户端的数据
```

以位置区分参数的RPC调用：

``` javascript
--> {"jsonrpc": "2.0", "method": "substract", "params": [42, 23], "id": 1}
<-- {"jsonrpc": "2.0", "result": 19, "id": 1}

--> {"jsonrpc": "2.0", "method": "substract", "params": [23, 42], "id": 2}
<-- {"jsonrpc": "2.0", "result": -19, "id" 2}
```

以名称区分参数的RPC调用：

``` javascript
--> {"jsonrpc": "2.0", "method": "substract", "params": {"substrahend": 23, "minuend": 42}, "id": 3}
<-- {"jsonrpc": "2.0", "result": 19, "id":3}

--> {"jsonrpc": "2.0", "method": "substract", "params": {"minuend": 42, "substrahend": 23}, "id": 4}
<-- {"jsonrpc": "2.0", "result": 19, "id": 4}
```

消息：

``` javascript
--> {"jsonrpc": "2.0", "method": "update", "params": [1,2,3,4,5]}
--> {"jsonrpc": "2.0", "method": "foobar"}
```

RPC调用不存在的方法：

``` javascript
--> {"jsonrpc": "2.0", "method": "foobar", "id": "1"}
<-- {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": "1"}
```

无效JSON的RPC调用：

``` 
--> {"jsonrpc": "2.0", "method": "foobar, "params": "bar", "baz]
<-- {"jsonrpc": "2.0", "error": {"code": -32700, "message": "Parse error"}, "id": null}
```

无效请求对象的RPC调用：

``` javascript
--> {"jsonrpc": "2.0", "method": 1, "params": "bar"}
<-- {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null}
```

批量RPC调用，无效的JSON：

``` 
--> [
  {"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"},
  {"jsonrpc": "2.0", "method"
]
<-- {"jsonrpc": "2.0", "error": {"code": -32700, "message": "Parse error"}, "id": null}
```

使用空数组进行RPC调用：

``` javascript
--> []
<-- {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null}
```

单个无效的批量RPC调用（非空数组）：

``` javascript
--> [1]
<-- [
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null}
]
```

多个无效的批量RPC调用：

``` javascript
--> [1,2,3]
<-- [
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
  {"jsonrpc": "2.0", "error": {"code": -32600, "message": "Invalid Request"}, "id": null},
]
```

批量RPC调用：

``` javascript
--> [
  {"jsonrpc": "2.0", "method": "sum", "params": [1,2,4], "id": "1"},
  {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]},
  {"jsonrpc": "2.0", "method": "substract", "params": [42,23], "id": "2"},
  {"foo": "boo"}.
  {"jsonrpc": "2.0", "method": "foo.get", "params": {"name": "myself"}, "id": "5"},
  {"jsonrpc": "2.0", "method": "get_data", "id": "9"}
]
<-- [
  {"jsonrpc": "2.0", "result": 7, "id": "1"},
  {"jsonrpc": "2.0", "result": 19, "id": "2"},
  {"jsonrpc": "2.0", "error": {"code":-32600, "message": "Invalid Request"}, "id": null},
  {"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": "5"},
  {"jsonrpc": "2.0", "result": ["hello", 5], "id": "9"}
]
```

批量RPC调用（全部为通知）：

``` javascript
--> [
  {"jsonrpc": "2.0", "method": "notify_sum", "params": [1,2,4]},
  {"jsonrpc": "2.0", "method": "notify_hello", "params": [7]}
]
<-- //Nothing is returned for all notification batches
```

## 8 扩展
以 *rpc.* 开头的方法名被保留用作系统扩展，必须不能作为他用。每个系统扩展都由相应的文档定义。所有的系统扩展都是可选的。
