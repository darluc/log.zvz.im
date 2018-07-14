title: 使用 Go 语言学会 Tensorflow
date: 2018-07-15 00:51:03
tags:
- golang
- tensorflow
---

Tensorflow 并不是一个专门用于机器学习的库，相反的，它是一个通用的用于图计算的库。它的核心部分是用 C++ 实现的，同时还有其它语言的接口库。Go 语言版本的接口库与 Python 版本的并不一样，它不仅有助于我们使用 Go 语言调用 Tensorflow，同时有助于我们了解 Tensorflow 的底层实现。

![](https://ws1.sinaimg.cn/large/7327fe71gy1fsnt4f7jeej209a07uq3u.jpg)
<!-- more -->
## 接口库

Tensorflow 官方发布的代码库包含：

* C++ 源代码：Tensorflow 核心功能高层 & 底层操作的代码实现。
* Python 接口库 & Python 功能库：接口库是通过 C++ 代码自动生成的，这样我们可以使用 Python 直接调用到 C++ 的方法：numpy 核心代码也是这样实现的。功能库则是对接口库方法的组合调用，它实现了大家所熟知的高层 API 接口。
* Java 接口库
* Go 接口库

我作为一名 Go 开发者，且不是 Java 爱好者，很自然地选择了使用 Go 版本的接口库，研究它能完成哪些任务。

## Go 接口库

首件值得注意的事，正如它的维护者们承认的，就是 Go 接口库缺少对 `变量` 支持：这些接口被设计成用于**使用**训练好的模型，而不是从零开始**训练**模型。这在 [Installing Tensorflow for Go ](https://www.tensorflow.org/versions/master/install/install_go) 中交待得很清楚。

> Tensorflow 提供了 Go 程序接口。这些接口特别适于加载 Python 库所创建的模型，然后在 Go 应用中执行。

如果我们对于训练机器学习模型不那么感兴趣：那就恰好！不过，若你对训练模型感兴趣的话，这里有一点建议：

> 作为一名真正的 Go 爱好者，应当寻求便宜之道！请使用 Python 来定义和训练模型；之后，你总是能用 Go 来加载并使用它们的。

简言之：Go 接口库可以用来**导入并定义**常量图；这里说的「常量」是指没有训练过程参与，所以没有可用于训练的变量。

让我们立刻开始用 Go 来调用 Tensorflow：创建我们的第一个应用程序。

接下来，我假设你们已经安装了 Go 环境，并且已经按照 [README](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/go/README.md) 编译并安装了 Tensorflow 的接口库。

## 理解 Tensorflow 的数据结构

我要在这里重申一下 Tensorflow 的定义（我为大家从 [Tensorflow 站点](https://www.tensorflow.org/)的说明中划出了重点）：

> TensorFlow™ 是一个使用数据流图进行数值计算的开源软件库。图中的节点**代表**数学操作，而图中的边则**代表**节点间相互联系的多维数据数组（张量）。

我们可以把 Tensorflow 看作是一种描述性语言，类似于 SQL，你可以用它描述你的需求，让底层引擎（数据库）解析你的 query 语句，检查语法和语义错误，将其转化为它的内部描述，优化并计算出结果：最后返回给你正确的结果。

所以，我们使用 API 接口时，实际是在描述一个图：当我们将它放入一个 `Session` 中，并且开始 `Run` 时，图的求值过程就开始了。

理解这些之后，让我们尝试定义一个计算图，并且在一个 `Session` 中计算它。[API 文档](https://godoc.org/github.com/tensorflow/tensorflow/tensorflow/go)能为我们清楚地提供  `tensorflow` （缩写 `tf` ）和 `op` 包的方法列表。

如你所见，这两个包包含了我们对图进行定义和计算所需要的一切。

前一个包包含了构建类似 `Graph` 本身等基础「空」结构的方法，后一个则是最重要的包，它包含了从 C++ 实现里自动生成的接口方法。

假设我们想要计算矩阵 A 和 x 的乘积：

$ A = \begin{pmatrix}1 & 2\\\\-1 &-2\end{pmatrix}, x = \begin{pmatrix}10\\\\100\end{pmatrix} $

我假设读者已经知道 tensorflow 图定义的概念，知道什么是占位符而且知道它们如何工作。下面的代码是用户第一次使用 Python 接口时可能会做的尝试。我们将其命名为 `attempt1.go`

```go
package main

import (
	"fmt"
	tf "github.com/tensorflow/tensorflow/tensorflow/go"
	"github.com/tensorflow/tensorflow/tensorflow/go/op"
)

func main() {
	// 让我们描述我们的需求：创建图

	// 我们想要定义两个运行时使用的 placeholder
	// 第一个 placeholder A 是 [2, 2] 整数张量
	// 第二个 placeholder x 是 [2, 1] 整数张量

	// 然后计算 Y = Ax

	// 创建图的节点：一个空节点，作为图的根节点
	root := op.NewScope()

	// 定义两个占位符
	A := op.Placeholder(root, tf.Int64, op.PlaceholderShape(tf.MakeShape(2, 2)))
	x := op.Placeholder(root, tf.Int64, op.PlaceholderShape(tf.MakeShape(2, 1)))

	// 定义可以接受 A & x 作为输入的操作节点
	product := op.MatMul(root, A, x)

	// 每次我们将 `Scope` 传入一个操作时，我们都将这个操作置于这个作用域内。
	// 如你所见，我们有一个通过 NewScope 创建空域：
	// 这个空域是我们所创建的图的根，我们用 “/”表示它。

	// 现在我们让 tensorflow 通过我们的定义来构建图。
	// 实体的图是通过我们用域和操作组合起来定义的“抽象”图生成的。

	graph, err := root.Finalize()
	if err != nil {
		// 处理这个错误没有什么用处
		// 如果我们对图的定义做错了，我们只能手动修正这些定义。

		// 它很想一个 SQL 查询过程：如果查询语句错了，我们只能重写它
		panic(err.Error())
	}

	// 至此：我们的图定义语法上就没有问题了。
	// 我们现在可以将其放入一个 Session 中使用了。

	var sess *tf.Session
	sess, err = tf.NewSession(graph, &tf.SessionOptions{})
	if err != nil {
		panic(err.Error())
	}

	// 为了使用占位符，我们必须创建含有数值的张量传入网络中
	var matrix, column *tf.Tensor

	// A = [ [1, 2], [-1, -2] ]
	if matrix, err = tf.NewTensor([2][2]int64{ {1, 2}, {-1, -2} }); err != nil {
		panic(err.Error())
	}
	// x = [ [10], [100] ]
	if column, err = tf.NewTensor([2][1]int64{ {10}, {100} }); err != nil {
		panic(err.Error())
	}

	var results []*tf.Tensor
	if results, err = sess.Run(map[tf.Output]*tf.Tensor{
		A: matrix,
		x: column,
	}, []tf.Output{product}, nil); err != nil {
		panic(err.Error())
	}
	for _, result := range results {
		fmt.Println(result.Value().([][]int64))
	}
}
```

代码内的注释非常丰富，请大家仔细阅读每行注释。

如果是 Python 版 Tensorflow 的使用者，现在已经可以期待代码编译后能完美运行了。我们看看是否能如愿呢：

`go run attempt1.go`

会得到如下结果：

`panic: failed to add operation "Placeholder": Duplicate node name in graph: 'Placeholder'`

稍等，这里发生了什么？错误提示很明显，有两个同名的占位符都叫作“PlaceHolder“。

## 第一课：节点 ID

使用 Python 接口时，每当我们调用定义操作的方法时，无论它是否已经被调用过，都会生成不同的节点。下面的代码就会很顺利的返回结果 3。

```python
import tensorflow as tf
a = tf.placeholder(tf.int32, shape=())
b = tf.placeholder(tf.int32, shape=())
add = tf.add(a,b)
sess = tf.InteractiveSession()
print(sess.run(add, feed_dict={a: 1,b: 2}))
```

要验证这段程序创建了两个不同的节点，我们只需要将占位符的名字打印出来：`print(a.name, b.name)` 输出 `Placeholder:0 Placeholder_1:0` 。 这里 `b` 占位符的名字是 `Placeholder_1:0` 同时 `a` 占位符的名字是 `Placeholder:0` 。

在 Go 版本里，则不同，之前程序就因为 `A` 和 `x` 都叫作 `Placeholder` 而导致运行失败。我们可以总结如下：

**Go 语言版 API 接口每次在我们调用定义操作的方法时，不会自动为节点生成新的名称**：操作名称是固定的，而且我们没法改变它。

**问答时间：**

* 关于 Tensorflow 系统我们学到了什么？对于一个图来说，它的每一个节点都必须有唯一的名称。节点是以各自的名字来区分的。
* 节点名称是否与定义它的操作名称相同？是的，更确切地讲，不完全是，只是名称的结尾部分相同。

为了说明第二个答案，让我们来修复节点的重名问题。

## 第二课：作用域

正如我们刚才看到，Python 版的 API 接口会在每次定义操作时，自动生成一个新的名字。从底层实现来看，Python 接口调用了 C++ 的 `Scope` 类的 `WithOpName` 方法。以下是此方法的文档和形式声明，来自 [scope.h](https://github.com/tensorflow/tensorflow/blob/a5b1fb8e56ceda0ee2794ee05f5a7642157875c5/tensorflow/cc/framework/scope.h) 头文件：

```c++
/// Return a new scope. All ops created within the returned scope will have
/// names of the form <name>/<op_name>[_<suffix].
Scope WithOpName(const string& op_name) const;
```

我们可以注意到这个用于命名节点的方法，其返回值是一个 `Scope` 对象，由此一个节点的名称，实际上是一个 `Scope` 域对象。一个 `Scope` 是一个**完整路径**，从根 `/` （空图）起到 `op_name` 结束。

当我们增加一个从`/` 到 `op_name` 有相同路径的节点时，会导致在同一个域中的节点重复，此时 `WithOpName` 方法会为名称添加一个后缀 `_<suffix>` （`<suffix>` 是一个计数器）。

知道这些以后，我们期望找到 `Scope 类型` 的  `WithOpName` 方法，来解决重复节点的问题。可惜的是，这个方法暂时还没有实现。

取而代之的，在[文档中的 Scope 类型](https://godoc.org/github.com/tensorflow/tensorflow/tensorflow/go/op#Scope)部分我们看到唯一能够返回一个新的 `Scope` 的方法是 `SubScope(namespace string)` 。

引用文档如下：

> 调用 SubScope 方法会返回一个新的 Scope，使得所有加入图中的操作都被置于命名空间 ‘namespace’ 中。如果命名空间与作用域中已有的命名空间重名，则会加上后缀。

使用后缀进行冲突管理与在 C++ 中使用 `WithOpName` 方法**不同**：`WithOpName` 在同一个作用域内的操作名称后加上 `suffix` 后缀（这样 `Placeholder` 就变成了 `Placeholder_1` ），而 Go 使用的 `SubScope` 的方法则是**对作用域名称**增加后缀名 `suffix` 。

这点差异会产生完全不同的图，不过尽管不同（节点放在不同的作用域中），从计算角度看它们是等价的。

让我们修改一下占位符的定义过程，定义两个不同的节点，然后打印出 `Scope` 的名称。

让我们创建文件 `attempt2.go` 将下面几行代码

```go
A := op.Placeholder(root, tf.Int64, op.PlaceholderShape(tf.MakeShape(2, 2)))
x := op.Placeholder(root, tf.Int64, op.PlaceholderShape(tf.MakeShape(2, 1)))
```

改成

```go
// define 2 subscopes of the root subscopes, called "input". In this
// way we expect to have a input/ and a input_1/ scope under the root scope
A := op.Placeholder(root.SubScope("input"), tf.Int64, op.PlaceholderShape(tf.MakeShape(2, 2)))
x := op.Placeholder(root.SubScope("input"), tf.Int64, op.PlaceholderShape(tf.MakeShape(2, 1)))
	fmt.Println(A.Op.Name(), x.Op.Name())
```

正常编译并运行：`go run attempt2.go` 。结果如下：

```shell
input/Placeholder input_1/Placeholder
```

**问答时间：**

关于 Tensorflow 系统我们学到了什么？一个节点可由它被定义的作用域所区分。作用域就是从图的根节点直到操作节点的路径。有两种方式可以定义执行相同操作的节点：在不同的作用域中定义操作（Go 的方式）或者改变操作名称（Python 自动实现或者我们可以使用 C++ 做到）

我们刚刚解决了节点名称重复的问题，另一个问题又出现了。

```shell
panic: failed to add operation "MatMul": Value for attr 'T' of int64 is not in the list of allowed values: half, float, double, int32, complex64, complex128
```

为什么 `MatMul` 节点定义会报错？我们只是想让两个 `tf.int64` 矩阵相乘！看起来 `int64` 是 `MatMul` 唯一不能接受的参数类型。

> 属性 ‘T’ 的取值 int64，不在允许的列表中：half，float，double，int32，complex32， complex64， complex128

这是什么列表？为什么我们可以将两个 `int32 ` 类型的矩阵相乘却不支持 `int64` 类型？

让我们继续研究这个问题，搞清楚到底发生了什么。

## 第三课：Tensorflow 类型体系

让我们深入到 C++ 源码中，看一下 `MatMul` 操作的函数声明：

```c++
REGISTER_OP("MatMul")
    .Input("a: T")
    .Input("b: T")
    .Output("product: T")
    .Attr("transpose_a: bool = false")
    .Attr("transpose_b: bool = false")
    .Attr("T: {half, float, double, int32, complex64, complex128}")
    .SetShapeFn(shape_inference::MatMulShape)
    .Doc(R"doc(
Multiply the matrix "a" by the matrix "b".
The inputs must be two-dimensional matrices and the inner dimension of
"a" (after being transposed if transpose_a is true) must match the
outer dimension of "b" (after being transposed if transposed_b is
true).
*Note*: The default kernel implementation for MatMul on GPUs uses
cublas.
transpose_a: If true, "a" is transposed before multiplication.
transpose_b: If true, "b" is transposed before multiplication.
)doc");
```

这行代码定义了 `MatMul` 操作的接口：特别注意，我们使用 `REGISTER_OP` 宏声明了操作的：

* 名称：`MatMul`
* 参数：`a`，`b`
* 属性（可选参数）：`transpose_a`，`transpose_b`
* 模板 `T` 支持的类型：`half, float, double, int32, complex64, complex128`
* 输出形式：自动推理的
* 文档

这个宏调用不包含任何 C++ 代码，不过它告诉我们**当定义个一个操作时，即使它使用了模板，我们也必须指定对于指定类型（或属性）`T` 所支持的类型列表**。例如，属性 `.Attr("T: {half, float, double, int32, complex64, complex128}")` 就限制了类型 `T` 必须是列表中的某一项。

我们可以从[教程](https://www.tensorflow.org/extend/adding_an_op)中看到，甚至在使用模板 `T` 的时候，我们也必须为每个支持的重载显示地注册到内核中。内核是以 CUAD 方式对 C/C++ 函数进行并行调用执行的。

`MatMul` 的作者之所以决定只支持之前列出的参数类型，而不支持 `int64` 类型，可能有以下两个原因：

1. 疏忽：这是有可能的，毕竟 Tensorflow 的代码也是人写的！
2. 为了支持那些不完全支持 `int64` 类型操作的设备，这样内核的这些特定实现就不会到处都是，而导致在本可以支持的硬件上无法运行。

回到我们的报错上来：修复的方法很明显。我们必须要给 `MatMul` 方法传递它所支持的数据类型。

让我们创建 `attempt3.go` 文件，将每一行用到 `int64` 的地方改成 `int32`。

有件事要注意一下：**Go 语言的接口包定义了一套自有的类型，与 Go 原生类型基本上是 1:1 对应的关系。当我们向图内填入参数时需要对照这个对应关系（比如，对于定义为 `tf.Int32` 的占位符要传入 `int32` 类型的值）。从图中读取数据时也要准从相同的法则。**由张量计算返回的`*tf.Tensor` 类型，自带 `Value()` 方法，它可以返回一个 `interface{}` 类型的值，必须由我们去转化为正确的类型（我们构建图的时候可知此类型）。

正常执行 `go run attempt3.go` 。结果如下：

```shell
input/Placeholder input_1/Placeholder
[[210] [-210]]
```

棒极了！

这儿有一份完整的 `attempt3` 的代码，你可以编译并运行它。

```go
package main                                        

import (                                            
        "fmt"                                       
        tf "github.com/tensorflow/tensorflow/tensorflow/go"                                              
        "github.com/tensorflow/tensorflow/tensorflow/go/op"                                              
)                                                   

func main() {                                       
        // Let's describe what we want: create the graph                                                 

        // We want to define two placeholder to fill at runtime                                          
        // the first placeholder A will be a [2, 2] tensor of integers                                   
        // the second placeholder x will be a [2, 1] tensor of intergers                                 

        // Then we want to compute Y = Ax           

        // Create the first node of the graph: an empty node, the root of our graph                      
        root := op.NewScope()                       

        // Define the 2 placeholders                
        // define 2 subscopes of the root subscopes, called "input". In this                             
        // way we expect the have a input/ and a input_1/ scope under the root scope                     
        A := op.Placeholder(root.SubScope("input"), tf.Int32, op.PlaceholderShape(tf.MakeShape(2, 2)))   
        x := op.Placeholder(root.SubScope("input"), tf.Int32, op.PlaceholderShape(tf.MakeShape(2, 1)))   
        fmt.Println(A.Op.Name(), x.Op.Name())       

        // Define the operation node that accepts A & x as inputs                                        
        product := op.MatMul(root, A, x)            

        // Every time we passed a `Scope` to an operation, we placed that operation **under**            
        // that scope.                              
        // As you can see, we have an empty scope (created with NewScope): the empty scope               
        // is the root of our graph and thus we denote it with "/".                                      

        // Now we ask tensorflow to build the graph from our definition.                                 
        // The concrete graph is created from the "abstract" graph we defined using the combination      
        // of scope and op.                         

        graph, err := root.Finalize()               
        if err != nil {                             
                // It's useless trying to handle this error in any way:                                  
                // if we defined the graph wrongly we have to manually fix the definition.               

                // It's like a SQL query: if the query is not syntactically valid we have to rewrite it  
                panic(err.Error())                  
        }                                           

        // If here: our graph is syntatically valid.
        // We can now place it within a Session and execute it.

        var sess *tf.Session                        
        sess, err = tf.NewSession(graph, &tf.SessionOptions{})                                           
        if err != nil {                             
                panic(err.Error())                  
        }                                           

        // In order to use placeholders, we have to create the Tensors containing the values to feed into                                                                                                          
        // the network                              
        var matrix, column *tf.Tensor               

        // A = [ [1, 2], [-1, -2] ]                 
        if matrix, err = tf.NewTensor([2][2]int32{{1, 2}, {-1, -2}}); err != nil {                       
                panic(err.Error())                  
        }                                           
        // x = [ [10], [100] ]                      
        if column, err = tf.NewTensor([2][1]int32{{10}, {100}}); err != nil {                            
                panic(err.Error())                  
        }                                           

        var results []*tf.Tensor                    
        if results, err = sess.Run(map[tf.Output]*tf.Tensor{                                             
                A: matrix,                          
                x: column,                          
        }, []tf.Output{product}, nil); err != nil { 
                panic(err.Error())                  
        }                                           
        for _, result := range results {            
                fmt.Println(result.Value().([][]int32))                                                  
        }                                           
}
```

**问答时间：**

关于 Tensorflow 系统我们学到了什么？每个操作都有它自己的关联核心实现。Tensorflow 可以看作是一种强类型的描述性语言。它不仅要遵守 C++ 的类型规则，它还得在注册操作时指定执行时使用数据的类型。

## 总结

使用 Go 语言定义图并进行运算，带给我们一次深入理解 Tensorflow 底层架构的机会。采取逐步试错的方式，我们解决了这个简单的问题，而且一步步学习到了关于图，图的节点以及类型体系的新知识。





翻译自：[Understanding Tensorflow using Go](https://pgaleone.eu/tensorflow/go/2017/05/29/understanding-tensorflow-using-go/)

