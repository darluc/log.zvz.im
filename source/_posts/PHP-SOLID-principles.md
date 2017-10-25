title: 面向对象设计的五个首要原则
date: 2017-10-17 14:39:51
tags:
- php
---

**S.O.L.I.D** 是 Robert C. Martin（人称 [Bob 大叔](https://en.wikipedia.org/wiki/Robert_Cecil_Martin)） 提出的**面向对象设计五个首要原则**的首字母缩写。

将这些原则组合运用起来，可以使程序员开发出的代码更易于维护和扩展。它们还可以避免开发者写出糟糕的代码，使代码更易于重构，同时它们也是敏捷或适应性软件开发方式的组成部分。

**注意**：本文只是一篇“欢迎使用 **S.O.L.I.D**”的入门文章，简单地说明了 **S.O.L.I.D** 是什么。

**S.O.L.I.D 代表着：**

把缩写被展开时，看起似乎麻烦了一点，实际上很容易掌握。

* **S** - Single-responsibility priciple 单一职责原则
* **O** - Open-closed priciple 开放闭合原则
* **L** - Liskov substitution principle 里氏替换原则
* **I** - Interface segregation principle 接口隔离原则
* **D** - Dependency inversion principle 依赖反转原则

接下来让我们挨个研究每条原则，以理解为什么 S.O.L.I.D 能帮我们成为更好的开发者。
<!-- more -->

## 单一职责原则

简写为 **S.R.P** - 此原则的描述如下：

> 一个类应当有且仅有一个原因进行更改，也就是说一个类应当只有一项任务。

举例来说，假设我们有一些图形，我们想要对这些图形的面积求和。这是个很简单的任务吧？

```php
class Circle {
    public $radius;

    public function __construct($radius) {
        $this->radius = $radius;
    }
}

class Square {
    public $length;

    public function __construct($length) {
        $this->length = $length;
    }
}
```

首先，我们创建了图形类，并且通过构造函数设定了必需的参数。接下来，我们创建了 **AreaCalculator** 类，写出对图形面积求和的逻辑。

```php
class AreaCalculator {

    protected $shapes;

    public function __construct($shapes = array()) {
        $this->shapes = $shapes;
    }

    public function sum() {
        // logic to sum the areas
    }

    public function output() {
        return implode('', array(
            "",
                "Sum of the areas of provided shapes: ",
                $this->sum(),
            ""
        ));
    }
}
```

要使用 **AreaCalculator** 类，我们只需简单地初始化这个类，并传入包含一组图形的数组，最后打印出输出信息。

```php
$shapes = array(
    new Circle(2),
    new Square(5),
    new Square(6)
);

$areas = new AreaCalculator($shapes);

echo $areas->output();
```

此处的 output 方法的问题在于 **AreaCalculator** 类处理了数据输出的逻辑，如果用户想要输出 json 或者其它格式时，我们该如何应对？

将所有这些逻辑都交由 **AreaCalculator** 类来处理，恰恰是与单一职责原则相悖的；**AreaCalculator** 类应当只负责计算所提供图形的面积之和，而无需关心用户需要 json 还是 HTML。

所以，为了解决这个问题，你可以创建一个 **SumCalculatorOutputter** 类，用于处理图形面积之和的显示逻辑。

**SumCalculatorOutputter** 类的用法大体如下：

```php
$shapes = array(
    new Circle(2),
    new Square(5),
    new Square(6)
);

$areas = new AreaCalculator($shapes);
$output = new SumCalculatorOutputter($areas);

echo $output->JSON();
echo $output->HAML();
echo $output->HTML();
echo $output->JADE(); 
```

现在，无论你需要什么样的输出显示逻辑，全部都交给 **SumCalculatorOutputter** 类来处理了。

## 开放闭合原则

> 对象或者实体应当对于扩展保持开放，对于修改保持关闭。

简单来说就是一个类应当易于扩展，而不需要对自身进行修改。我们看一下 **AreaCalculator** 类，特别留意它的 **sum** 方法。

```php
public function sum() {
    foreach($this->shapes as $shape) {
        if(is_a($shape, 'Square')) {
            $area[] = pow($shape->length, 2);
        } else if(is_a($shape, 'Circle')) {
            $area[] = pi() * pow($shape->radius, 2);
        }
    }

    return array_sum($area);
}   
```

如果我们想扩展 **sum** 方法，使其能够对更多类型的图形进行面积求和，那么我们可以通过增加 **if/else 代码块**来解决这个问题，但是这样就破坏了开放闭合原则。

有一种办法可以对 **sum** 方法进行优化，就是去除 sum 方法中计算每种图形面积的逻辑，并将这些逻辑放到对应的图形类中。

```php
class Square {
    public $length;

    public function __construct($length) {
        $this->length = $length;
    }

    public function area() {
        return pow($this->length, 2);
    }
}
```

对 **Circle** 类也做同样的修改，加入一个 **area** 方法。现在，要计算任意类型图形的面积之和可以简单地实现为：

```php
public function sum() {
    foreach($this->shapes as $shape) {
        $area[] = $shape->area();
    }

    return array_sum($area);
}
```

现在，我们可以创建另一个的图形类，将其加入到求和计算中，而且不会搞坏代码。然而，另一个问题产生了，我们如何能够确定传入到 **AreaCalculator** 的对象是一个图形对象，且这个图形对象有这个 **area** 方法？

面向接口编程是 **S.O.L.I.D** 实践中的必需部分，以下是个简单的示例，我们创建了一个接口，所有的图形类都要实现该接口：

```php
interface ShapeInterface {
    public function area();
}

class Circle implements ShapeInterface {
    public $radius;

    public function __construct($radius) {
        $this->radius = $radius;
    }

    public function area() {
        return pi() * pow($this->radius, 2);
    }
}
```

在 **AreaCalculator** 类的 sum 方法中，我们可以检查这些图形是否实现了 **ShapeInterface** 接口，若未实现就抛出异常：

```php
public function sum() {
    foreach($this->shapes as $shape) {
        if(is_a($shape, 'ShapeInterface')) {
            $area[] = $shape->area();
            continue;
        }

        throw new AreaCalculatorInvalidShapeException;
    }

    return array_sum($area);
}
```

## 里氏替换原则

> 若 **q(x)** 可证明对象 **x** 的类型为 **T** 。那么 **q(y)** 应当可用于证明对象 **y** 的类型为 **S**，其中**S** 是 **T** 的子类型。

所有这些是说所有的子类或衍生类应当能够被它们的基类或父类替换。

继续以之前的 **AreaCalculator** 类为例，假设有一个 **VolumeCalculator** 类继承了 **AreaCalculator**：

```php
class VolumeCalculator extends AreaCalulator {
    public function __construct($shapes = array()) {
        parent::__construct($shapes);
    }

    public function sum() {
        // logic to calculate the volumes and then return and array of output
        return array($summedData);
    }
}
```

在 **SumCalculatorOutputter** 类中：

```php
class SumCalculatorOutputter {
    protected $calculator;

    public function __constructor(AreaCalculator $calculator) {
        $this->calculator = $calculator;
    }

    public function JSON() {
        $data = array(
            'sum' => $this->calculator->sum();
        );

        return json_encode($data);
    }

    public function HTML() {
        return implode('', array(
            '',
                'Sum of the areas of provided shapes: ',
                $this->calculator->sum(),
            ''
        ));
    }
}
```

如果我们尝试执行下面的代码：

```php
$areas = new AreaCalculator($shapes);
$volumes = new AreaCalculator($solidShapes);

$output = new SumCalculatorOutputter($areas);
$output2 = new SumCalculatorOutputter($volumes);
```

程序并不会报错，不过当我们调用 **$output2** 对象的 **HTML** 方法时，会得到一个 **E_NOTICE** 报错，提示我们将数组转为了字符串。

为了解决这个问题，我们让 **VolumeCalculator** 的 sum 方法不再返回数组，简单地修改为：

```php
public function sum() {
    // logic to calculate the volumes and then return and array of output
    return $summedData;
}
```

这个返回的「和」是一个浮点数或者整数。

## 接口隔离原则

> 不能强迫对象实现一个无用的接口，或者说对象不应当被强制依赖于它们不会使用到的方法。

接着看图形的例子，我们知道还有一些图形是立体的，所以我们也想计算出图形的体积，我们可以在 **ShanpeInterface** 中加入另一个接口方法：

```php
interface ShapeInterface {
    public function area();
    public function volume();
}
```

这样依赖，我们创建的任何图形类都得实现 **volume** 方法，但是我们知道正方形是平面图形，它们没有体积，所以这个接口会迫使 **Square** 类实现一个对它来说毫无用处的方法。

接口隔离原则禁止这种实现方式，相反地，你应该创建另一个名为 **SolidShapeInterface** 的接口，它拥有 **volume** 方法，正方体等立体图形可以实现这个接口：

```php
interface ShapeInterface {
    public function area();
}

interface SolidShapeInterface {
    public function volume();
}

class Cuboid implements ShapeInterface, SolidShapeInterface {
    public function area() {
        // calculate the surface area of the cuboid
    }

    public function volume() {
        // calculate the volume of the cuboid
    }
}
```

这种方式好得多。不过有一个陷阱需要注意，作为管理方法的参数类型提示时，不应该单独使用 **ShapeInterface** 或者 **SolidShapeInterface**。

你可以创建另一个接口，叫作 **ManageShapeInterface**，无论是平面图形还是立体图形都实现此接口，这样你就可以用单一接口管理这些图形了。例如：

```php
interface ManageShapeInterface {
    public function calculate();
}

class Square implements ShapeInterface, ManageShapeInterface {
    public function area() { /*Do stuff here*/ }

    public function calculate() {
        return $this->area();
    }
}

class Cuboid implements ShapeInterface, SolidShapeInterface, ManageShapeInterface {
    public function area() { /*Do stuff here*/ }
    public function volume() { /*Do stuff here*/ }

    public function calculate() {
        return $this->area() + $this->volume();
    }
}
```

这样在 **AreaCalculator** 类中，我们可以简单地使用 **calculate** 方法替换 **area** 方法，同时检查对象是否实现了 **ManageShapeInterface** 接口而不是 **ShapeInterface** 接口。

## 依赖反转原则

> 对象必须依赖于抽象，而不是依赖于具体实现。高层模块不应该依赖于低层模块，它们都应当依赖于抽象。

虽然讲起来一大堆，其实很好理解。此原则主要是考虑到解耦的问题，举例说明似乎是解释该原则最好的办法：

```php
class PasswordReminder {
    private $dbConnection;

    public function __construct(MySQLConnection $dbConnection) {
        $this->dbConnection = $dbConnection;
    }
}
```

首先，**MySQLConnection** 是低层模块而 **PasswordReminder** 是高层模块，根据依赖反转原则所述*依赖抽象而不是实现*，这段代码已经违背了这一原则，因为 **PasswordReminder** 类强依赖于 **MySQLConnection** 类。

其次，当你要更换数据库引擎时，你还得修改 **PasswordReminder** 类，这又违背了**开放闭合原则**。

**PasswordReminder** 类不需要关心你的应用具体使用哪一种数据库，要解决此问题我们还是采用“面向接口编程”的办法，由于高层和低层模块应当依赖于抽象，所以我们可以创建一个接口：

```php
interface DBConnectionInterface {
    public function connect();
}
```

此接口包含一个 connect 方法，**MySQLConnection** 类实现了这个接口，而且在 **PasswordReminder** 类的构造函数中，我们将 **MySQLConnection** 的类型提示改为此接口类型，这样一来无论你的应用使用何种数据库，**PasswordReminder** 都可以轻松连接数据库不会产生任何问题，也不会违背**开放闭合原则**。

```php
class MySQLConnection implements DBConnectionInterface {
    public function connect() {
        return "Database connection";
    }
}

class PasswordReminder {
    private $dbConnection;

    public function __construct(DBConnectionInterface $dbConnection) {
        $this->dbConnection = $dbConnection;
    }
}
```

依照上面的代码，你可以看到现在无论高层还是低层模块都变成依赖于抽象了。

## 总结

老实说，**S.O.L.I.D** 起初看起来好像有很多规则，但是在长期使用并遵守它的规则后，它会逐渐成为你的一部分，融入到你的代码中，使你的代码易于扩展，适应修改，容易测试并且重构起来没有任何麻烦。



翻译自：[S.O.L.I.D: The First 5 Principles of Object Oriented Design](https://scotch.io/bar-talk/s-o-l-i-d-the-first-five-principles-of-object-oriented-design)



