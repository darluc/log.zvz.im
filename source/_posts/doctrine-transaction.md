title: Doctrine transaction
date: 2015-09-02 02:29:35
tags:
- php
- doctrine
---
# Doctrine 2 ORM 2: Transaction Demarcation

## 事务划界

“事务划界”就是定义你的事务边界的。正确地进行事务划界非常重要，如果做的不好就会影响你的应用性能。许多数据库以及数据库抽象层（比如PDO）默认工作在自动提交（auto-commit）模式下，该模式下每条SQL语句都被包裹在一个单独的小事务中。所以如果你没有主动进行事务控制，很快就会造成应用的性能下降，因为事务的开销可不便宜哦。

在绝大多数情况下，Doctrine 2都为你做了合理的事务划界：所有的写操作（INSERT/UPDATE/DELETE）都被放到队列中，直到 `EntityManager#flush()`被调用，这些操作都被囊括在一个事务中。

而且，Doctrine 2也支持（且鼓励）你自己来接管并控制事务划界。

这里有使用Doctrine ORM处理事务的两种方法，接下来将详细讲述。
<!--more-->

### 方法1：隐式

第一种方法就是使用Doctrine ORM默认提供的事务处理机制。下面的代码片段，就没有任何显示的事务划界：

``` php
<?php
// $me instanceof EntityManager
$user = new User;
$user->setName('George');
$em->persist($user);
$em->flush();
```

上面的代码我们没有进行任何事务操作，`EntityManager#flush()`会启动一个事务，并且控制它的提交和回滚。Doctrine ORM以将数据操作语句聚合（the aggregation of the DML operations）的方式实现了此种事务划界方法。如果所有的数据操作都属于某个域模型（domain model）的一个工作单元，这种方法完全能够满足需求。

### 方法2：显式

显式的方式，是调用`Doctrine\DBAL\Connection` API直接控制事务边界。代码如下：

``` php
<?php
// $em instanceof EntityManager
$em->getConnetion()->beginTransaction(); // suspend auto-commit
try{
  //... do some work
  $user = new User;
  $user->setName('George');
  $em->persist($user);
  $em->flush();
  $em->getConnection()->commit();
}catch(Exception $e){
  $em->getConnection()->rollback();
  throw $e;
}
```

显式事务划界在两种情况下是必不可少的：

1. 当你想要在一个工作单元中包含自定义的DBAL操作时；
2. 当你想要使用某些需要激活状态事务的`EntityManager`API时。如果你没有满足它的事务需求时，这类方法会抛出`TransactionRequiredException`异常。

一个更加方便的显式事务划界方法，是使用控制抽象层的方法`Connection#transactional($func)`或`EntityManager#transactional($func)` 。这些控制抽象可以确保你不会遗漏事务回滚的处理，而且节省不少代码。下面的代码示例与前面的代码实现了同样的功能：

``` php
<?php
// $em instanceof EntityManager
$em->transactional(function($em){
  //... do some work
  $user = new User;
  $user->setName('George');
  $em->persist($user);
})
```

<p style='background-color:#FFFFDD;'>Notice: 由于历史原因。使用`EntityManager#transactional($func)` 时，如果函数`$func` 返回等同于false的值时（如`array()`， `"0"` ，`""` ，`0` ），总是会返回`false` 。</p>

`Connection#transactional` 和`EntityManager#transactional` 的区别在于，后者在提交事务前会清空（flush）`EntityManager` ，并且在发生异常时回滚事务。

### 异常处理

当使用隐式事务划界时，如果在执行`EntityManager#flush()`时发生异常 ，事务会被自动回滚，`EntityManager`会关闭。

当使用显式事务划界时，事务应当如上面的例子所展示的那样，立即回滚，同时调用`Entitymanger#close()`方法关闭`EntityManager`，随后删除之。当然也能像前例所示，使用控制抽象更加优雅地处理。注意一般情况下，你应当将捕获到的异常重新抛出。如果你想要从某些异常中恢复，可以将捕获这些异常的代码放在考前的位置（但不能忘记回滚事务并关闭`EntityManager`）。

在这些过程处理完成后，所有之前`EntityManager`管理过或移除掉的实例都脱离了（become detached）。这些脱线对象的状态与事务回滚时的状态是一致的，并且这种状态是无法回滚的，也就是说它们与数据库不再同步了。我们可以继续使用这些对象，但是得留意它们的状态可能已不准确了。

异常发生后，如果你还想进行下一个工作单元，那么你应当使用一个新的`EntityManger`对象。


translated from [Doctrine 2 ORM 2 documentation](http://doctrine-orm.readthedocs.org/en/latest/reference/transactions-and-concurrency.html)
