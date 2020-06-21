title: Go:ent，基于图的 ORM 框架 - Facebook 出品
date: 2020-06-21 23:57:22
tags:
- golang
---

![](https://img.zvz.im/imgs/2020/06/85d574b9bcb0a790.png)

:information_source: *本文基于处于开发中的项目的主分支*

`ent` 是由 [Facebook Connectivity](https://connectivity.fb.com/) 团队创建的 ORM 框架。迫于 Go 社区中缺少能够像图一样查询数据的工具，同时也缺少 100% 类型安全的 ORM，`ent` [就是被设计出来解决这些问题的](https://entgo.io/blog/#the-motivation-for-writing-a-new-orm-in-go)。
<!-- more -->
## 图的概念

`ent` 用一组实体和边线来表达图的概念，并且没有任何限制。我们使用代码库中现成的例子进行说明：

![](https://img.zvz.im/imgs/2020/06/0b1995b25318a023.png)

此例子中，有三个实体对象，`Group` ，`User` ，`Pet` ，它们相互关联在一起，可以用多种方式遍历。这个图可以利用项目中的 `entc` 工具来描述，使用命令 `entc describe` ：

![](https://img.zvz.im/imgs/2020/06/aedee6eb2dbe9289.png)

实体之间的关系由边线表示，它们在内部被转换为字段，表和外键。这里是在 MySQL 中转换成的数据库图表：

![](https://img.zvz.im/imgs/2020/06/1d8e45619c1eb5ef.png)

传统的 ORM 框架把每个关系显示地申明在实体上，与此不同，`ent` 只暴露了实体的属性和一个用于遍历这些实体边线的方法。

## 查询

这个项目提供了优秀的文档和各种不同 ORM 查询和使用的例子。让我们来看一个涉及所有实体的查询例子：

*从**管理员**用户的朋友中，我们想要获得他们的宠物的朋友的主人*。

以下是这个查询的示意图：

![](https://img.zvz.im/imgs/2020/06/e7be54884261b2e8.png)

例子中建议的查询是可自我描述的：

![](https://img.zvz.im/imgs/2020/06/523d81b91c4b287e.png)

如上一段所示，每个实体都暴露了方法用来顺着它们的边线进行查询：`Group` 通过 `QueryAdmin()` 方法暴露了 `admin` 边线，它自己又通过 `QueryFriends()` 暴露了它的 `friends` 边线，等等。代码生成是用来创建这些方法的核心工具。

## 代码生成

通过 `entc` 命令可以进行代码生成，它是项目的核心部件而且它使这个项目更加强大。模板引擎是代码生成的基础，它允许开发者修改查询语句，并依照他们的需求暴露各种方法，使得 ORM 足够灵活能够任意地定制。

默认的实体模板会提供所有的文件和方法用于创建、更新、删除和遍历查询每个实体对象以及它们的关系边。无论如何，在代码生成中一切都是模板。你可以制定它与数据库的交流方式，数据迁移等等。

这个项目还提供了一个实现了 GraphQL [ `Node` 接口](https://facebook.github.io/relay/graphql/objectidentification.htm#sec-Node-Interface)的模板示例。生成的代码使得我们的实体对象都能兼容这个接口：

![](https://img.zvz.im/imgs/2020/06/5f78378a3b0e63a2.png)

模板系统能生成完美契合你的项目的代码。

## 性能

现在来到有趣的部分了，看看底层的查询是如何构建的。这里以 MySQL 为例，上例中生成的查询如下：

![](https://img.zvz.im/imgs/2020/06/5ca7c6e0b64b64c4.png)

这个 ORM 使用嵌套查询来实现无限制地遍历查询。不过使用 join 查询也是可行的。这是与嵌套查询等效的使用 join 的查询：

![](https://img.zvz.im/imgs/2020/06/cb0b33f8e0554d78.png)

虽然嵌套的查询更容易读，由于是一层层的 select 语句嵌套出来的，大多数情况下，[join 语句执行更快](https://dev.mysql.com/doc/refman/5.7/en/rewriting-subqueries.html)。不过，在这个案例中，由于 ORM 遍历数据使用的边线是相关的索引，服务器生成的查询计划是一样的：

![plan for the subqueries](https://img.zvz.im/imgs/2020/06/0b0b1ba8d0a653f6.png)

<center><small>plan for the subqueries</small></center>

![plan for the join](https://img.zvz.im/imgs/2020/06/e987261aedfa0a0a.png)

<center><small>plan for the join</small></center>

这个 ORM 框架还支持更复杂的查询通过图来查询节点，比如：

*我们想要获得所有的宠物，它们的朋友的主人是 `admin` 成员组的朋友*。

这个查询的 SQL 写起来非常难。使用常规的 ORM 框架几乎是个不可能完成的任务，不过用 `ent` 写起来却非常简单：

![](https://img.zvz.im/imgs/2020/06/35c48fae4fe6da94.png)

而且， `ent` 不需要像传统 ORM 框架一样对多个领域的多个节点进行聚合，所以性能总会不错。对查询做一个快速的跑分，可以看出代码包增加了多少额外的工作：

```bash
name        time/op
Traverse-8   179µs ± 5%

name        alloc/op
Traverse-8  21.4kB ± 0%

name        allocs/op
Traverse-8     414 ± 0%
```

现在，我们不对数据库进行实际查询，再跑一遍：

```bash
name        time/op
Traverse-8  64.4µs ± 1%
```

 数据库查询本身占用了前一个跑分的大部分时间。项目本身引入的延迟很低，这保证了它会有一个优秀的前景。想要知道更多的开发细节，此项目已经[在 Github 放出了线路图](https://github.com/facebookincubator/ent/issues/46)。





> 翻译自：[Go: ent, Graph-Based ORM by Facebook](https://medium.com/a-journey-with-go/go-ent-graph-based-orm-by-facebook-d9ba6d2290c6)

