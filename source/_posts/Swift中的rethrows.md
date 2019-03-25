---
title: Swift中的rethrows
date: 2016-05-03 14:59:11
tags: Swift
category: Swift
thumbnailImage: http://d.image.i4.cn/i4web/image/ueditor/php/upload/image/20140723/1406108151591224.jpg
thumbnailImagePosition: right
autoThumbnailImage: yes
---
我最近在学习Swift函数式编程时，越来越觉得Swift是一门强大的语言。在 Swift 的世界中，函数不再是二等公民。是的，Swift 引入了大量函数式编程的特性，使得我们能够把函数当作一等公民来对待。在Swift中，适当引入函数式编程的思想和方法，常常会有奇效。
<!--more-->
然而，当我想去深入了解时，发现这里水好深，还有好多自己不知道的东西。
废话不多说，我们先从`map`函数说起吧。Swift中`map`是这么声明的：
```swift
 public func map<T>(@noescape transform: (Self.Generator.Element) throws -> T) rethrows -> [T]
```
这里`@noescape`是什么东西？`rethrows `又是什么东西？查了资料才知道原来是这么回事儿：
`@noescape`，这是一个从 Swift 1.2 引入的关键字，它是专门用于修饰函数闭包这种参数类型的，当出现这个参数时，它表示该闭包不会跳出这个函数调用的生命期：即函数调用完之后，这个闭包的生命期也结束了。以下是苹果的文档原文：
>A new @noescape attribute may be used on closure parameters to functions. This indicates that the parameter is only ever called (or passed as an @noescape parameter in a call), which means that it cannot outlive the lifetime of the call. This enables some minor performance optimizations, but more importantly disables the self. requirement in closure arguments.

如果想了解更多关于`@noescape`，可以看看这篇文章：http://nshint.io/blog/2015/10/23/noescape-attribute/

那`rethrows `又是怎么一回事儿呢？下面我们就通过写一个我们自己的`map`来看一看`rethrows `是个什么鬼。
```swift
extension Array {
    func mymap<T>(@noescape transform: (Generator.Element) -> T) -> [T] {
        var ts = [T]()
        for e in self {
            ts.append(transform(e))
        }
        return ts
    }
}

enum CalculationError: ErrorType {
    case DivideByZero
}

func squareOf(x: Int) -> Int {return x*x}

func divideTenBy(x: Int) throws -> Double {
    guard x != 0 else {
        throw CalculationError.DivideByZero
    }
    return 10.0 / Double(x)
}

```
下面我们来调用一下`mymap `函数：


![屏幕快照 2016-03-01 上午9.54.43.png](http://upload-images.jianshu.io/upload_images/1351863-2bc93aaf0c7719b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到当我们传人的闭包有异常抛出时，编译器就报错了。根据报错信息我们重写了个`map `函数:
```swift
func mymapThrow<T>(@noescape transform: (Generator.Element) throws -> T) throws -> [T] {
        var ts = [T]()
        for e in self {
            ts.append(try transform(e))
        }
        return ts
    }
    
```
来调用一下`mymapThrow `函数：

![屏幕快照 2016-03-01 上午10.09.11.png](http://upload-images.jianshu.io/upload_images/1351863-eb9c916287489d77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
编译器又报错了，我们让新的`map`函数能抛出异常，然后在调用`mymapThrow `函数的地方处理异常。但是这会带来一个问题，例如`x2`这里我们传入的闭包并没有异常抛出啊，难道我们在每次调用的时候都非得写那么一大串异常处理的代码吗？例如这样：
```swift
let ns: [Double]
do {
    try ns = xs.mymapThrow(divideTenBy)
    ns
} catch {
    
}

let ns2: [Double]
do {
    try ns2 = xs.mymapThrow(squareOf)
} catch {
    
}
```
按 Swift 类型安全的写法，在有异常抛出的地方就一定需要使用 try 语法。我相信在平时我们传入的闭包函数没有异常的情况一定远远多于有异常的情况，难道我们非得为了代码的安全性就必须牺牲掉方便性吗？显然，Swift比我们想象的要更聪明。于是本文章的主角`rethrows`登场了。
我们重新写个`map`函数：
```swift
func _map<T>(@noescape transform: (Generator.Element) throws -> T) rethrows -> [T] {
        var ts = [T]()
        for e in self {
            ts.append(try transform(e))
        }
        return ts
    }
```
再来看一下结果：

![屏幕快照 2016-03-01 上午10.28.18.png](http://upload-images.jianshu.io/upload_images/1351863-c837b571719e6495.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这一下就没问题了。当传入的闭包函数没有异常时我们也不用去捕获异常，有异常时我们就去处理异常。所以`rethrows`关键字的意义就在于：
>这个函数如果抛出异常，仅可能是因为传递给它的闭包的调用导致了异常。如果闭包的调用没有导致异常，编译器就知道这个函数不会抛出异常。那么我们也就不用去处理异常了。

哈哈，一个`map`函数就有这么多的学问，看来我还得花更多的精力去学习Swift了。

