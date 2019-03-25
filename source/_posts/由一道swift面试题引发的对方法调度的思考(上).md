---
title: 由一道swift面试题引发的对方法调度的思考
date: 2017-02-14 14:36:34
tags: Swift
category: Swift
thumbnailImage: https://www.raizlabs.com/dev/wp-content/uploads/sites/10/2016/12/Summary-3-768x380.png
thumbnailImagePosition: bottom
autoThumbnailImage: yes
---
最近在看swift面试题时，其中有一道题目让我很诧异。题目是这样的：

以下代码会打印出什么？
<!--more-->
```swift
protocol Pizzeria { 
  func makePizza(_ ingredients: [String])
  func makeMargherita()
} 

extension Pizzeria { 
  func makeMargherita() { 
    return makePizza(["tomato", "mozzarella"]) 
  }
}

struct Lombardis: Pizzeria { 
  func makePizza(_ ingredients: [String]) { 
    print(ingredients)
  } 
  func makeMargherita() {
    return makePizza(["tomato", "basil", "mozzarella"]) 
  }
}

let lombardis1: Pizzeria = Lombardis()
let lombardis2: Lombardis = Lombardis() 
lombardis1.makeMargherita()
lombardis2.makeMargherita()
```
当然，即使是swift新手也会毫不犹豫的给出答案：


打印两行`["tomato", "basil", "mozzarella"]`

然后面试官笑了笑，将`Pizzeria`中声明的`makeMargherita()`去掉，代码变为：
```swift
protocol Pizzeria { 
  func makePizza(_ ingredients: [String])
} 

extension Pizzeria { 
  func makeMargherita() { 
    return makePizza(["tomato", "mozzarella"]) 
  }
}

struct Lombardis: Pizzeria { 
  func makePizza(_ ingredients: [String]) { 
    print(ingredients)
  } 
  func makeMargherita() {
    return makePizza(["tomato", "basil", "mozzarella"]) 
  }
}

let lombardis1: Pizzeria = Lombardis()
let lombardis2: Lombardis = Lombardis() 
lombardis1.makeMargherita()
lombardis2.makeMargherita()
```
估计有很多童鞋会跟我一样，不假思索地给出答案：打印两行`["tomato", "basil", "mozzarella"]`。如果答案还是一样，面试官就没有删除那行代码的必要了吧。正确答案应该是：
```
["tomato", "mozzarella"]
["tomato", "basil", "mozzarella"]
```
聪明的童鞋即使不知道正确答案，知道此处有陷阱，也会给出了正确答案。那么导致这种结果的真正原因是什么呢？答案就是**方法调度(Method Dispatch)**
## 什么是方法调度
方法调度就是一个程序在调用一个方法时如何选择要执行的指令的过程。当我们每次调用一个方法时方法调度都会发生。

编译型语言有三种基础的方法调度方式: 直接调度(Direct Dispatch), 函数表调度(Table Dispatch) 和 消息调度(Message Dispatch)。大部分语言支持一到两种。Java默认使用函数表调度，你可以通过使用 `final` 关键字将其变为直接调度。C++默认使用直接调度，通过 `virtual` 关键字可以改为函数表调度。Objective-C总是使用消息调度。但允许开发者使用C直接派发来获取性能的提高。Swift在这方面走在了前面，她支持全部的3种调度方式。这样的方式非常好,，不过也给很多Swift开发者带来了困扰。

## 调度类型（Types of Dispatch）

调度的目的是程序告诉CPU被调用的函数在哪里，在我们深入Swift的这种行为之前，有必要了解一下方法调度的三种方式。

**直接调度(Direct Dispatch)**

直接调度是最快的, 不止是因为需要调用的指令集会更少, 并且编译器还能够有很大的优化空间, 例如函数内联等, 但这不在这篇博客的讨论范围。

然而, 对于编程来说直接调用也是最大的局限, 而且因为缺乏动态性所以没办法支持继承。

**函数表调度 (Table Dispatch )**

函数表调度是编译型语言实现动态行为最常见的实现方式. 函数表使用了一个数组来存储类声明的每一个函数的指针. 大部分语言把这个称为 “virtual table”(虚函数表), Swift 里称为 “witness table”. 每一个类都会维护一个函数表, 里面记录着类所有的函数, 如果父类函数被 `override` 的话, 表里面只会保存被 `override` 之后的函数. 一个子类新添加的函数, 都会被插入到这个数组的最后. 运行时会根据这一个表去决定实际要被调用的函数.

看看下面的例子：
```swift
class ParentClass {
    func method1() {}
    func method2() {}
}
class ChildClass: ParentClasss {
    override func method2() {}
    func method3() {}
}
```
在这个情况下, 编译器会创建两个函数表, 一个是 `ParentClass` 的, 另一个是 `ChildClass` 的:

![](https://www.raizlabs.com/dev/wp-content/uploads/sites/10/2016/12/virtual-dispatch-768x227.png)

```swift
let obj = ChildClass()
obj.method2()
```
当一个方法被调用时，会经历下面几个过程：

1. 读取 `0xB00` 对象的调度表
2. 通过索引读取该方法的函数指针，在这里, `method2` 的索引是1(偏移量), 所以地址就是 `0xB00 + 1`
3. 跳到 `0x222` (函数指针指向 0x222)

查表是一种简单, 易实现, 而且性能可预知的方式. 然而, 这种派发方式比起直接派发还是慢一点。从字节码角度来看, 多了两次读和一次跳转, 由此带来了性能的损耗。另一个慢的原因在于编译器可能会由于函数内执行的任务导致无法优化。

这种基于数组的实现, 缺陷在于函数表无法拓展。子类会在虚数函数表的最后插入新的方法, 没有位置可以让 extension 安全地插入函数。

**消息调度 (Message Dispatch )**

消息调度是调用函数最动态的方式。也是 Cocoa 的基石, 这样的机制催生了 [KVO](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html), [UIAppearence](https://developer.apple.com/reference/uikit/uiappearance) 和 [CoreData](https://developer.apple.com/library/content///documentation/Cocoa/Conceptual/CoreData/index.html) 等功能. 这种运作方式的关键在于开发者可以在运行时改变函数的行为. 不止可以通过 [swizzling](https://www.mikeash.com/pyblog/friday-qa-2010-01-29-method-replacement-for-fun-and-profit.html) 来改变, 甚至可以用 [isa-swizzling](http://stackoverflow.com/questions/38877465/are-method-swizzling-and-isa-swizzling-the-same-thing/38878119#38878119) 修改对象的继承关系, 可以在面向对象的基础上实现自定义调度。

看下面两个类:
```swift
class ParentClass {
    dynamic func method1() {}
    dynamic func method1() {}
}
class ChildClass: ParentClass {
    override func method2() {}
    dynamic func method3() {}
}
```
Swift 会用树来构建这种继承关系:

![](https://www.raizlabs.com/dev/wp-content/uploads/sites/10/2016/12/message-dispatch-768x412.png)

当一个消息被发送时, 运行时会顺着类的继承关系向上查找应该被调用的方法. 如果你觉得这样做效率很低, 它确实很低! 然而, 只要缓存建立了起来, 这个查找过程就会通过缓存来把性能提高到和函数表一样快. 但这只是消息机制的原理, [这里有一篇文章](http://www.friday.com/bbum/2009/12/18/objc_msgsend-part-1-the-road-map/)很深入的讲解了具体的技术细节.

## Swift 的调度机制
那么，swift是如何调度的呢？这里有四个方面,来指导如何选择调度:

* 方法声明的位置
* 引用类型
* 特定的行为
* 显式地优化

要说明的是Swift 并没有在文档里具体写明什么时候会使用函数表什么时候使用消息机制. 唯一的承诺是使用 `dynamic` 修饰的时候会通过 Objective-C 的运行时使用消息机制。

**声明的位置 (Location Matters)**

在Swift中有两个地方可以声明一个方法：类型声明的作用域内和 `extension`。根据声明类型的不同, 也会有不同的派发方式:
```swift
class MyClass {
    func mainMethod() {}
}
extension MyClass {
    func extensionMethod() {}
}
```
上面的例子里, `mainMethod` 会使用函数表的方式, 而 `extensionMethod` 则会使用直接调度。根据声明的位置，可以总结如下：

![](https://www.raizlabs.com/dev/wp-content/uploads/sites/10/2016/12/Defaults-1-768x503.png)

总结起来有这么几点:

* 值类型总是会使用直接派发, 简单易懂
* 而协议和类的 `extension` 都会使用直接调度
* 协议和普通Swift类声明作用域里的方法都会使用函数表进行调度
* 继承 `NSObject` 的类声明作用域里的方法都会使用函数表调度
* 继承 `NSObject` 的类的 `extension` 会使用消息调度

**引用类型 (Reference Type Matters)**

引用的类型决定了调度的方式, 这是显而易见的, 但有一个重要的区别。 一个比较常见的疑惑, 发生在一个协议拓展和类型拓展同时实现了同一个函数的时候。
```swift
protocol Animal {
}
extension Animal {
  func extensionMethod() {
    print("In Protocol extension method")
  }
}

struct 🐱: Animal {
}
extension 🐱 {
  func extensionMethod() {
    print("喵喵")
  }
}

let cat = 🐱()
let proto: Animal = cat

cat.extensionMethod()
proto.extensionMethod()
```
刚接触 Swift 的童鞋可能会认为 `proto.extensionMethod() `调用的是结构体里的实现。 但是, 引用的类型决定了调度的方式, 协议拓展里的方法会使用直接调度。如果把 `extensionMethod` 的声明移动到协议的声明位置的话, 则会使用函数表调度, 最终就会调用结构体里的实现。 并且要记得, 如果两种声明方式都使用了直接调度的话, 基于直接调度的运作方式, 我们不可能实现预想的 `override` 行为。



