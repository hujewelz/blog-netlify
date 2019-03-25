---
title: 从 Swift 中的序列到类型擦除
date: 2018-01-06 11:52:43
tags: Swift
categories: Swift
coverImage: Swift_logo.png
---

如果有这样的一个需求，我希望能像数组一样，用 for 循环遍历一个类或结构体中的所有属性。要实现这样的需求，我们需要让自定义的类型遵守 Sequence 协议。

<!-- excerpt -->

如果有这样的一个需求，我希望能像数组一样，用 for 循环遍历一个类或结构体中的所有属性。就像下面这样：

```Swift
let persion = Persion()

for i in persion {
    print(i)
}
```

要实现这样的需求，我们需要让自定义的类型遵守 Sequence 协议。



## 序列

Sequence 协议是集合类型结构中的基础。一个序列 (sequence) 代表的是一系列具有相同类型的值，你可以对这些值进行迭代。Sequence 协议提供了许多强大的功能，满足该协议的类型都可以直接使用这些功能。上面这样步进式的迭代元素的能力看起来十分简单，但它却是 Sequence 可以提供这些强大功能的基础。



满足 Sequence 协议的要求十分简单，你需要做的所有事情就是提供一个返回迭代器 (iterator) 的 `makeIterator() ` 方法：

```Swift
public protocol Sequence {
    associatedtype Iterator : IteratorProtocol
    
    public func makeIterator() -> Self.Iterator
    
    // ...
}
```

在 Sequence 协议有个关联类型 Iterator，而且它必须遵守 IteratorProtocol 协议。从这里我们可以看出 Sequence 是一个可以创建迭代器协议的类型。所以在搞清楚它的步进式的迭代元素能力之前，有必要了解一下迭代器是什么。


## 迭代器

序列通过创建一个迭代器来提供对元素的访问。迭代器每次产生一个序列的值，并且当遍历序列时对遍历状态进行管理。在 IteratorProtocol 协议中唯一的一个方法是 next()，这个方法需要在每次被调用时返回序列中的下一个值。当序列被耗尽时，next() 应该返回 nil，不然迭代器就会一直工作下去，直到资源被耗尽为止。

IteratorProtocol 的定义非常简单：

```Swift
public protocol IteratorProtocol {
    associatedtype Element
    
    public mutating func next() -> Self.Element?
}
```

关联类型 Element 指定了迭代器产生的值的类型。这里`next()` 被标记了 mutating，表明了迭代器是可以存在可变的状态的。这里的 mutating 也不是必须的，如果你的迭代器返回的值并没有改变迭代器本身，那么没有 mutating 也是没有任何问题的。 不过几乎所有有意义的迭代器都会要求可变状态，这样它们才能够管理在序列中的当前位置。

对 Sequence 和 IteratorProtocol 有了基础了解后，要实现开头提到的需求就很简单了。比如我想迭代输出一个 Person 实例的所有属性，我们可以这样做：

```Swift
struct Persion: Sequence {
    var name: String
    var age: Int
    var email: String
    
    func makeIterator() -> MyIterator {
        return MyIterator(obj: self)
    }
}
```

Persion 遵守了 Sequence 协议，并返回了一个自定义的迭代器。迭代器的实现也很简单：

```Swift
struct MyIterator: IteratorProtocol {
    var children: Mirror.Children
    
    init(obj: Persion) {
        children = Mirror(reflecting: obj).children
    }
   
    mutating func next() -> String? {
        guard let child = children.popFirst() else { return nil }
        return "\(child.label.wrapped) is \(child.value)"
    }
}
```

迭代器中的 `children` 是 `AnyCollection<Mirror.Child>`  的集合类型，每次迭代返回一个值后，更新 `children` 这个状态，这样我们的迭代器就可以持续的输出正确的值了，直到输出完 `children` 中的所有值。

现在可以使用 for 循环输出 Persion 中所有的属性值了：

```Swift
for item in Persion.author {
    print(item)
}

// out put:
// name is jewelz
// age is 23
// email is hujewelz@gmail.com
```

如果现在有另外一个结构体或类也需要迭代输出所以属性呢？，这很好办，让我们的结构体遵守 Sequence 协议，并返回一个我们自定义的迭代器就可以了。这种拷贝代码的方式确实能满足需求，但是如果我们利用协议拓展就能写出更易于维护的代码，类似下面这样：

```swift
struct _Iterator: IteratorProtocol {
    var children: Mirror.Children
    
    init(obj: Any) {
        children = Mirror(reflecting: obj).children
    }
    
    mutating func next() -> String? {
        guard let child = children.popFirst() else { return nil }
        return "\(child.label.wrapped) is \(child.value)"
    }
}

protocol Sequencible: Sequence { }

extension Sequencible {
    func makeIterator() -> _Iterator {
        return _Iterator(obj: self)
    }
}
```

这里我定义了一个继承 Sequence 的空协议，是为了不影响 Sequence 的默认行为。现在只要我们自定义的类或结构体遵守 Sequencible 就能使用 for 循环输出其所有属性值了。就像下面这样：

```swift
struct Demo: Sequencible {
    var name = "Sequence"
    var author = Persion.author
}
```


## 表示相同序列的类型 

现在需求又变了，我想将所有遵守了 Sequencible 协议的任何序列存到一个数组中，然后 for 循环遍历数组中的元素，因为数组中的元素都遵守了 Sequencible 协议，所以又可以使用 for 循环输出其所有属性，就像下面这样：

```Swift
for obj in array {
    for item in obj {
        print(item)
    }
}
```

那么这里的 array 应该定义成什么类型呢？定义成 [Any] 类型肯定是不行的，这样的话在循环中得将 item 强转为 Sequencible，那么是否可以定义成  [Sequencible] 类型呢？答案是否定的。当这样定义时编辑器会报出这样的错误：

{% alert danger %}

Protocol 'Sequencible' can only be used as a generic constraint because it has Self or associated type requirements

{% endalert %}

熟悉 Swift 协议的同学应该对这个报错比较熟了。就是说含有 Self 或者关联类型的协议，只能被当作泛型约束使用。所以像下面这样定义我们的 array 是行不通的。

```Swift
let sequencibleStore: [Sequencible] = [Persion.author, Demo()]
```

如果有这样一个类型，可以隐藏 Sequencible 这个具体的类型不就解决这个问题了吗？这种将指定类型移除的过程，就被称为类型擦除。


## 类型擦除

回想一下  Sequence 协议的内容，我们只要通过 `makeIterator()` 返回一个迭代器就可以了。那么我们可以实现一个封装类(结构体也是一样的)，里面用一个属性存储了迭代器的实现，然后在 `makeIterator()` 方法中通过存储的这个属性构造一个迭代器。类似这样：

```Swift
func makeIterator() -> _AnyIterator<Element> {
    return _AnyIterator(iteratorImpl)
}
```

我们的这个封装可以这样定义：

```Swift
struct _AnySequence<Element>: Sequence {
    private var iteratorImpl: () -> Element?
}
```

对于刚刚上面的那个数组就可以这样初始化了：

```Swift
let sequencibleStore: [_AnySequence<String>] = [_AnySequence(Persion.author), _AnySequence(Demo())]
```

这里的 _AnySequence 就将具体的 Sequence 类型隐藏了，调用者只知道数组中的元素是一个可以迭代输出字符串类型的序列。

现在我们可以一步步来实现上面的 \_AnyIterator 和 \_AnySequence。_AnyIterator 的实现跟上面提到的 _AnySequence 的思路一致。我们不直接存储迭代器，而是让封装类存储迭代器的 next 函数。要做到这一点，我们必须首先将 iterator 参数复制到一个变量中，这样我们就可以调用它的 next 方法了。下面是具体实现：

```Swift
struct _AnyIterator<Element> {
    var nextImpl: () -> Element?
}

extension _AnyIterator: IteratorProtocol {
    init<I>(_ iterator: I) where Element == I.Element, I: IteratorProtocol {
        var mutatedIterator = iterator
        nextImpl = { mutatedIterator.next() }
    }
    
    mutating func next() -> Element? {
        return nextImpl()
    }
}
```

现在，在 \_AnyIterator 中，迭代器的具体类型（比如上面用到的\_Iterator）只有在创建实例的时候被指定。在那之后具体的类型就被隐藏了起来。我们可以使用任意类型的迭代器来创建 \_AnyIterator 实例：

```swift
var iterator = _AnyIterator(_Iterator(obj: Persion.author))
while let item = iterator.next() {
    print(item)
}
// out put:
// name is jewelz
// age is 23
// email is hujewelz@gmail.com
```

我们希望外面传入一个闭包也能创建一个 _AnyIterator，现在我们添加下面的代码：

```swift
 init(_ impl: @escaping () -> Element?) {
     nextImpl = impl
 }
```

添加这个初始化方法其实为了方便后面实现  \_AnySequence 用的。上面说过 \_AnySequence 有个属性存储了迭代器的实现，所以我们的 _AnyIterator 能通过一个闭包来初始化。

_AnyIterator 实现完后就可以来实现我们的 \_AnySequence 了。我这里直接给出代码，同学们可以自己去实现：

```swift
struct _AnySequence<Element> {

    typealias Iterator = _AnyIterator<Element>
    
    private var iteratorImpl: () -> Element?
}

extension _AnySequence: Sequence {
    init<S>(_ base: S) where Element == S.Iterator.Element, S: Sequence {
        var iterator = base.makeIterator()
        iteratorImpl = {
            iterator.next()
        }
    }
    
    func makeIterator() -> _AnyIterator<Element> {
        return _AnyIterator(iteratorImpl)
    }
}
```

 \_AnySequence 的指定构造器也被定义为泛型，接受一个遵循 Sequence 协议的任何序列作为参数，并且规定了这个序列的迭代器的 next() 的返回类型要跟我们定义的这个泛型结构的 Element 类型要一致。这里的这个泛型约束其实就是我们实现类型擦除的魔法所在了。它将具体的序列的类型隐藏了起来，只要序列中的值都是相同的类型就可以当做同一种类型来使用。就像下面的例子中的 array 就可以描述为 "元素类型是 String 的任意序列的集合"。

```swift
let array = [_AnySequence(Persion.author), _AnySequence(Demo())]

for obj in array {
    print("+-------------------------+")
    for item in obj {
        print(item)
    }
}
// out put:
// name is jewelz
//  age is 23
// email is hujewelz@gmail.com
// +-------------------------+
// name is Sequence
// author is Persion(name: "jewelz", age: 23, email: "hujewelz@gmail.com")
```

得益于 Swift 的类型推断，这里的 array 可以不用显式地指明其类型，点击 option 键，你会发现它是 `[_AnySequence<String>]` 类型。也就是说只有其元素是 String 的任意序列都可以作为数组的元素。这就跟我们平时使用类似 "一个 Int 类型的数组" 的语义是一致的了。如果要向数组中插入一个新元素，可以这样创建一个序列：

```swift
let s = _AnySequence { () -> _AnyIterator<String> in
    return _AnyIterator { () -> String? in
        return arc4random() % 10 == 5 ? nil : String(Int(arc4random() % 10))
    }
}
array.append(s)
```

上面的代码中通过一个闭包初始化了一个 _AnySequence，这里我就不给出自己的实现，同学们可以自己动手实现一下。


## 写在最后

在标准库中，其实已经提供了 **AnyIterator** 和 **AnySequence**。我还没去看标准库的实现，有兴趣的同学可以点击[这里](https://github.com/apple/swift/tree/master/stdlib/public/core)查看。 我这里实现了自己的 \_AnyIterator 和 \_AnySequence 就是为了提供一种实现类型擦除的思路。如果你在项目中频繁地使用带有关联类型或 Self 的协议，那么你也一定会遇到跟我一样的问题。这时候实现一个类型擦除的封装，将具体的类型隐藏了起来，你就不用为 Xcode 的报错而抓狂了。

