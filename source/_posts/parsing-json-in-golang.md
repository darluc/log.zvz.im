title: 如何使用 Go 语言解析 JSON
date: 2019-02-18 01:17:13
tags:
- golang
---

当静态编程语言使用到 JSON 的时候，总是会有些费力。一方面，JSON 数据可能是任何形式的，从一个简单的数字，到一个复杂的包含内嵌对象的数组。当使用 Go 语言的时候，这意味着你要将这些变化多端的 JSON 数据放入结构化的变量中去。

幸运地是，Go 尽力尝试帮我们降低编码难度，为我们提供了许多方式来解析 JSON 数据。

## 概述

Go 标准库的 json 包为我们提供了我们想要的功能。对于任意的 JSON 字符串，标准的解析方法如下：

```go
import "encoding/json"
//...

// ... 
myJsonString := `{"some":"json"}`

// `&myStoredVariable` is the address of the variable we want to store our
// parsed data in
json.Unmarshall([]byte(myJsonString), &myStoredVariable)
//...
```

本文中我们要讨论的是，对于 `myStoredVariable` 变量的类型，你可以有哪些不同的选择，以及决定何时采用何种类型。
<!--more-->

在处理 JSON 数据时，你会遇到两种情况：
1. 结构化数据
2. 非结构化数据

## 结构化数据

我们先从结构化数据开始，因为它们处理起来相对容易一些。对于这类数据我们需要提前知晓其结构。比如，你有一个 `Bird` 对象，每种鸟都有 `species` 字段和一个 `description` 字段：

```json
{
  "species": "pigeon",
  "decription": "likes to perch on rocks"
}
```

读取此类数据，只要创建一个 `struct` 结构体，作为你想要解析的数据的镜像。对于这个例子，我们会创建一个包含 `Species` 和 `Description` 字段的 `Bird` 结构体：

```go
type Bird struct {
  Species string
  Description string
}
```

然后将数据解析出来，如下：

```go
birdJson := `{"species": "pigeon","description": "likes to perch on rocks"}`
var bird Bird	
json.Unmarshal([]byte(birdJson), &bird)
fmt.Printf("Species: %s, Description: %s", bird.Species, bird.Description)
//Species: pigeon, Description: likes to perch on rocks
```

> 转换时，Go 不区分名称大小写问题，使用 JSON 属性与字段都转为小写后相同作为映射依据。所以， `Bird` 结构的 `Species` 字段会映射到名为 `species` ，或者 `Species` 或者 `sPeCiEs` 的 JSON 属性。

### JSON 数组

当遇到一个 `Bird` 数组的时候，又会发生什么呢？

```json
[
  {
    "species": "pigeon",
    "decription": "likes to perch on rocks"
  },
  {
    "species":"eagle",
    "description":"bird of prey"
  }
]
```

既然这个数组的每个元素都是一个 `Bird` ，你可以创建一个 `Bird` 类型的数组来解析它：

```go
birdJson := `[{"species":"pigeon","decription":"likes to perch on rocks"},{"species":"eagle","description":"bird of prey"}]`
var birds []Bird
json.Unmarshal([]byte(birdJson), &birds)
fmt.Printf("Birds : %+v", birds)
//Birds : [{Species:pigeon Description:} {Species:eagle Description:bird of prey}]
```

### 嵌套对象

现在，考虑这种情况，`Bird` 有一个 `Dimensions` 属性，用来描述它的 `Height` （高度）和 `Length` （身长）：

```json
{
  "species": "pigeon",
  "decription": "likes to perch on rocks",
  "dimensions": {
    "height": 24,
    "width": 10
  }
}
```

和上一个例子类似，我们需要将这个新的问题对象映射到我们的 Go 代码中。在机构体中加入一个内嵌的 `dimensions` 字段，让我们先申明一个 `dimensions` 的结构体类型：

```go
type Dimensions struct {
  Height int
  Width int
}
```

然后，`Bird` 结构体可以包含一个 `Dimensions` 字段：

```go
type Bird struct {
  Species string
  Description string
  Dimensions Dimensions
}
```

如此就能用与之前一样的方法进行数据解析了：

```go
birdJson := `{"species":"pigeon","description":"likes to perch on rocks", "dimensions":{"height":24,"width":10}}`
var birds Bird
json.Unmarshal([]byte(birdJson), &birds)
fmt.Printf(bird)
// {pigeon likes to perch on rocks {24 10}}
```

### 自定义属性名称

前面提到过 Go 会进行大小写转换来确定结构体字段和 JSON 属性的映射关系。不过很多时候，你会需要一个和 JSON 数据属性不同名的字段名称。

```json
{
  "birdType": "pigeon",
  "what it does": "likes to perch on rocks"
}
```

对于上面的 JSON 数据，我想让 `birdType` 属性映射到 Go 代码中的 `Species` 字段。同时我也没法给 `"what it does"` 提供一个合适的字段名称。

为了解决这些情况，我们可以使用结构体字段标签：

```go
type Bird struct {
  Species string `json:"birdType"`
  Description string `json:"what it does"`
}
```

现在，我们可以明确地告诉代码，某个 JSON 属性应该映射到哪个字段上了。

```go
birdJson := `{"birdType": "pigeon","what it does": "likes to perch on rocks"}`
var bird Bird
json.Unmarshal([]byte(birdJson), &bird)
fmt.Println(bird)
// {pigeon likes to perch on rocks}
```

## 非结构化数据

如果你遇到一些数据，它们的结构和属性名都不确定，那么你就无法使用结构体来解析这些数据了。取而代之地，你可以使用映射（maps）类型。考虑以下的 JSON 形式：

```json
{
  "birds": {
    "pigeon":"likes to perch on rocks",
    "eagle":"bird of prey"
  },
  "animals": "none"
}
```

我们没法构建一个结构体来描述上面的数据，因为鸟儿的名字作为对象的键值总是可以变换，而这会导致数据结构的变化。

为了处理这种情况，我们创建一个字符串对应空接口的 map：

```go
birdJson := `{"birds":{"pigeon":"likes to perch on rocks","eagle":"bird of prey"},"animals":"none"}`
var result map[string]interface{}
json.Unmarshal([]byte(birdJson), &result)

// The object stored in the "birds" key is also stored as 
// a map[string]interface{} type, and its type is asserted from
// the interface{} type
birds := result["birds"].(map[string]interface{})

for key, value := range birds {
  // Each value is an interface{} type, that is type asserted as a string
  fmt.Println(key, value.(string))
}
```

每个字符串对应一个 JSON 属性的名称， `interface{}` 类型对应它的值，这个值可以是任意类型。在代码中，其数据类型通过对 `interface{}` 进行断言就可以获取到。而且这些映射解析动作可以迭代执行，如此一来，可变数量的键通过一次循环中就能够处理完成。

## 如何选择

通常情况下，只要你能够使用结构体来描述你的 JSON 数据，你就应该使用它们。只有当遇到数据中有不确定的键或值时，无法使用结构体进行描述，才是你使用映射（map）的唯一理由。


翻译自：[Parsing JSON in Golang](https://www.sohamkamani.com/blog/2017/10/18/parsing-json-in-golang/)
