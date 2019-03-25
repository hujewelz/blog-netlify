---
title: Swift构造器
date: 2016-02-28 09:05:06
tags: Swift
category: Swift
---
构造过程就是为一个实例上的每个存储属性设置初始值，并在新实例准备就绪之前执行所需的任何其他设置或初始化。

<!--more-->

我们通过定义构造器（暂时就这么叫吧，因为大家都这么叫，其实我觉得称为初始化器或初始化方法更合适一点）来实现这个构造过程，其实它就是一个特殊的方法，可以用来创建一个特定类型的新示例。

Swift 中的构造器与 Objective-C 中不同，它没有返回值。不过它们的主要作用都是确保类型的新实例在第一次使用之前已正确初始化。OC 中我们并没有显示地给每个属性赋初始值，是因为它们在定义时有默认值，这一点与 Swift 不同。



## 为存储属性设置初始值
类和结构体必须在创建该类或结构体的实例时将其所有存储属性设置为适当的初始值。当然我们可以在定义属性时做好这个工作，不过大多数情况下，我们都是在构造器中给存储属性设置初始值。其实也并不是所有的存储属性都得去设置初始值，可选类型因为会被默认置为nil，所以并不强制在初始化时赋值。有一点需要注意的是不论是你在定义属性时就给定一个初值还是在构造器中给定初始值都不会触发属性监听。



### 构造器

构造器会在创建实例的时候自动调用，一般来说每个类都需要构造器，无论是自己写的还是编译器为你生成的。我们使用 `init` 关键字就可以定义一定构造器，并且不用使用 `func` 关键字:
```swift
init() {
}
```
我们可以在括号里面可以加各种参数，来进行更复杂的初始化。
```swift
var name: String
var age: Int

init(name: String, age: Int) {
    self.name = name
    self.age = age
}
```
和普通函数一样，构造函数也可以有内部参数名和外部参数名，在我们没有给定外部参数名的情况下，Swift 会自动给我们生成一个跟内部参数名一样的外部参数名。如果不想使用外部参数名，可以在内部参数名之前加上 `_` 即可。
```swift
init(_ name: String) {
}
```



## 默认构造器

如果在一个类或结构体中所有的存储属性都有默认值，并且没有定义任何构造器也没有父类，那么 Swift 就会提供一个默认的构造器。
```
class ShoppingListItem {
    var name: String?
    var quantity = 1
    var purchased = false
}
var item = ShoppingListItem()
```

### 结构体的逐一成员构造器
如果我们在结构体中没有定义任何自己的构造器，那么 Swift 会给我们提供一个逐一成员构造器，这和默认构造器不同，不管结构体中的存储属性有没有赋初识值，包括常量属性。
```swift

struct Size {
  var width: Float
  var height: Float
  
  let maxWH: Float
}

let aSize = Size(width: 200, height: 300, maxWH: 200)
```



## 值类型的构造器代理

所谓构造代理就是构造器可以调用别的构造器来辅助完成构造过程， 目的主要是为了避免写重复的代码。

值类型（结构体和枚举）和类的构造器代理规则是不一样的。值类型不支持继承，所以它们的构造器代理就比较简单，只需要通过 `self.init` 调用自己的其他构造器。下面是苹果官方给出的例子：
```swift
struct Size {
    var width = 0.0, height = 0.0
}
struct Point {
    var x = 0.0, y = 0.0
}

struct Rect {
    var origin = Point()
    var size = Size()
    init() {}
    init(origin: Point, size: Size) {
        self.origin = origin
        self.size = size
    }
    init(center: Point, size: Size) {
        let originX = center.x - (size.width / 2)
        let originY = center.y - (size.height / 2)
        self.init(origin: Point(x: originX, y: originY), size: size)
    }
}
```
如果给值类型定义了自己的构造器，那么默认的构造器将不会被合成，如果你还是想要默认的构造器，就用extension来写自定义构造器即可。



## 类的继承和初始化

类的所有存储属性包括从父类继承而来的都必须在初始化的时候赋予初值。Swift 为类定义了两种构造器来确保所有的存储属性都获得初值，即指定构造器和便利构造器。



### 指定构造器和便利构造器

指定构造器是主要的构造手段，每个类至少要有1个，意味着可以有多个，但是多数情况下会是1个，而便利构造器则没有要求。指定构造器必须完全初始化该类引入的所有存储属性，并调用适当的父类构造器以继续完成父类中属性的初始化过程。便利构造器则要求最终要调用一个指派构造器。

在语法上便利构造器和指定构造器没有太大差别，只是在指定构造器 `init` 前多了一个 `convenience` 关键字
``` swift
convenience init(params) {
}
```


### 类的构造器代理

为了简化指定构造器和便利构造器之间的关系，Swift 对构造器代理应用以下三个规则：

1. **一个指定构造器必须调用其直接父类的指定构造器。**
2. **一个便利构造器必须调用类中其它的构造器。**
3. **一个便利构造器必须最终调用一个指定构造器。**

用两句话概括就是：

* 指定构造器总是向上代理
* 便利构造器总是横向代理

下面的图很好地解释了这个规则
![](http://image18-c.poco.cn/mypoco/myphoto/20170228/13/1843604332017022813450909.png?487x379_130)



### 两段式构造

Swift 中类的初始化分为两个阶段。在第一个阶段中，每个存储属性都要被赋值，如果有一部分属性是从父类继承得来，那么调用父类的构造器来完成所有存储属性的赋值。反正就是在第一个阶段要保证所有的存储属性都要有初始值。第二阶段就可以对属性进行进一步操作了。

使用两段式构造会让初始化过程更加安全。两段式构造防止属性值在初始化之前被访问，并防止属性值被另一个构造器意外设置为不同的值。

同时，Swift 的编译器会为两段式构造执行四个有用的安全检查，以确保这个过程不会发生错误：

* **安全检查 1：在进行构造器的向上代理之前，必须确保类中所有的存储属性都被初始化。**

如上所述，当所有存储属性的初始状态已知时，对象的内存才被认为是完全初始化的。为了满足这条规则，指定构造器必须确保在它在移交给初始化链之前初始化它自己的所有属性。

* **安全检查 2：在给继承来的属性赋值之前，指定构造器必须先向上代理父类的构造器。如果不这么做的话，在给继承来的属性赋完值后，它可能会在父类的构造器中被重写。**
* **安全检查 3：一个便利构造器在给任何属性赋值之前，必须先调用其它的构造器。如果不这么做的话，便利构造器赋的值会被指定构造器覆盖。**
* **安全检查 4：在两段式构造过程的第一阶段完成之前，初始化函数不能调用其它实例方法, 不能从实例属性中取值, 也不能用`self`。**

结合上面的四个安全检查，再来看看两段式构造过程做了什么：

##### 阶段 1
* 类的一个指定构造器或便利构造器被调用。
* 为该类的新实例分配内存。但是内存还未初始化。
* 指定构造器确保所有的存储属性都有值。这些存储属性的内存现在已初始化。
* 指定构造器将初始化移交给父类初始化函数来对父类存储属性实现同样的操作。
* 这个过程一直沿着继承链持续下去，直到达到继承链顶端。
* 到了继承链顶端，并且最终父类保证所有的存储属性都有值之后，实例的内存就被当做完全初始化了，此时阶段1完成。

##### 阶段 2
* 从继承链顶端倒回来，每一个指派初始化函数都可以进一步定制实例，初始化函数至此可以访问 `self`，并且可以修改自己的属性，调用实例方法，等等。
* 最终，调用链上的任意便利构造器都可以操作实例了，也可以访问 `self`。



### 构造器的继承和重载

与 Objective-C 不同，Swift 中的子类默认不会自动从父类那里继承其构造器。虽然不能自动继承不过我们还是有办法从父类那里继承构造器。

当子类在覆盖父类的指定构造器时，需要要用 `override` 来修饰，即使你的子类的构造器的实现是一个便利构造器。但是，如果子类的指定造器与父类的便利构造器相同，那么父类的便利构造器永远都不会被子类调用到，所以这种情况是不需要 `override` 的。

```
class Animal {
  var numberOfFoots: Int
  var name: String
  
  init(name: String, foots: Int) {
    self.name = name
    numberOfFoots = foots
  }
  
  convenience init(name: String) {
    self.init(name: name, foots: 4)
  }
}

class 🐔: Animal {
  
  override init(name: String, foots: Int) {
    super.init(name: name, foots: foots)
    numberOfFoots = 2
  }
  
  init(name: String) {
    super.init(name: name, foots: 2)
  }
}
```
在上面的例子中，`Animal` 分别定义了一个指定构造器和便利构造器，在 🐔 中重写了父类的` init(name: String, foots: Int)`，所以要使用 `override` 关键字，并且又定义了一个跟父类便利构造器相同的指定构造器，这个时候就不需要 `override` 关键字了。因为便利构造器只能横向代理，是不会被子类重写的。



### 构造器的自动继承

上面已经说过，Swift 中的子类默认不会自动继承父类的构造函数。但是，如果满足某些条件，父类的构造器会自动继承。以下就是构造器会自动继承的两个规则：

* **规则 1** <br>
    如果子类没有定义任何指定构造器，它会自动继承父类所有的指定构造器。
* **规则 2** <br>
    如果子类实现了父类中所有的指定构造器——不管是从规则1继承而来还是提供了自己的实现，那么它会自动继承父类所有的便利构造器。

```swift
class Person {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
    
    convenience init(age: Int) {
      self.init(name: "unknow", age: age)
    }
    
    convenience init(name: String) {
        self.init(name: name, age: 0)
    }
}

class Man: Person {}
```
上面的代码中 `Man` 继承了 `Person` ，根据规则 1，`Man` 会继承父类的所有的指定构造器，同时根据规则 2，`Man` 实现了父类中所有的指定构造器 (这里是继承得来)，所以它会自动继承父类所有的便利构造器。就像下面这样：
![](http://image18-c.poco.cn/mypoco/myphoto/20170228/15/18436043320170228153534059.png?421x118_130)



## 可失败的构造器

有时候我们需要构造失败，例如传入了不恰当的参数，这种情况使用可失败的构造器是很有用的，这样可以引起调用者的注意。

为了应对可能失败的初始化条件，可以为我们的类型（类、结构体或枚举）定义一个或多个可失败构造器。用 `init?` 就可以定义一个可失败构造器。不过可失败和不可失败的构造器不能有一样的参数类型列表，否则就有二义性了。当构造失败时，可以返回 `nil`。
```swift
struct Animal {
    let species: String
    init?(species: String) {
        if species.isEmpty { return nil }
        self.species = species
    }
}

```
通过可失败构造器实例化的对象是个可选值，所以在使用时需要解包：
```swift
if let giraffe = Animal(species: "Giraffe") {
    print("An animal was initialized with a species of \(giraffe.species)")
}
```



### 可失败构造器的传递

类，结构或枚举的可失败构造器可以从相同的类，结构或枚举中代理给另一个可失败构造器。类似地，子类的可失败构造器可以向上代理父类的可失败构造器。

在任何种情况下，如果委托给另一个构造器导致初始化失败，整个初始化过程立即失败，并且不再执行初始化代码。
```swift
lass Product {
    let name: String
    init?(name: String) {
        if name.isEmpty { return nil }
        self.name = name
    }
}
 
class CartItem: Product {
    let quantity: Int
    init?(name: String, quantity: Int) {
        if quantity < 1 { return nil }
        self.quantity = quantity
        super.init(name: name)
    }
}

```



### 重载可失败构造器

你可以像重载普通构造器一样，重载父类的可失败构造器。你可以在子类中用普通的构造器重载父类的可失败构造器，但是不能反过来。看下面的例子就很清楚了：
```swift
class Document {
    var name: String?

    init() {}

    init?(name: String) {
        if name.isEmpty { return nil }
        self.name = name
    }
}

class AutomaticallyNamedDocument: Document {
    override init() {
        super.init()
        self.name = "[Untitled]"
    }
    override init(name: String) {
        super.init()
        if name.isEmpty {
            self.name = "[Untitled]"
        } else {
            self.name = name
        }
    }
}

class UntitledDocument: Document {
    override init() {
        super.init(name: "[Untitled]")!
    }
}

```



## 必需构造器(Required Initializers)

如果 `init` 用了 `required` 来修饰， 那么意味着子类必需要实现这个构造器。
```swift
class SomeClass {
    required init() {
        // initializer implementation goes here
    }
}

```
在子类重写父类的必需构造器时，仍然需要用`required` 来修饰，而不用使用 `override` 关键字。
```swift
class SomeSubclass: SomeClass {
    required init() {
        // subclass implementation of the required initializer goes here
    }
}

```
如果你满足构造器继承的条件的话，必需构造器也不是一定要自己实现的。



## 总结

以上就是 Swift 中构造器的简单介绍。与 Objective-C 相比，Swift 中的构造过程确实比较复杂，这也确确实实地体现了 Swift 是一门安全的语言。总的来说，Swift 的构造过程就是为了在任何实例使用之前所有的存储属性都要先初始化完成。

