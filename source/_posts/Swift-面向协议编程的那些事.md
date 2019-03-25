---
title: Swift与面向协议编程的那些事
date: 2018-01-08 11:53:41
tags: Swift
categories: Swift
---



一直想写一些 Swift 的东西，却不知道从何写起。因为想写的东西太多，然后所有的东西都混杂在一起，导致什么都写不出来。翻了翻以前在组内分享的一些东西，想想把这些内容整理下，写进博客吧。我对计划要写的东西做了个清单（最近做什么都喜欢在前一天睡觉前做个清单，这样多少改善了我的拖延症🤪）：

<!--more-->

- [ ] 面向协议编程
- [ ] 使用值类型代替引用类型
- [ ] 函数式编程
- [ ] 单向数据流



面向协议编程是 Swift 不同于其他语言的一个特性之一，也是比 Objective-C 强大的一个语言特性（并不是Swift 独有的，但是比 OC 的协议要强大很多），所以以面向协议编程作为 Swift 系列文章的开端是最合适不过的了。

文章的内容可能有点长，我就把要讲的内容简单地列了一下，同学们可以根据自己掌握的情况，跳到对应的小结进行阅读。下面是主要内容：

* 面向协议编程不是个新概念
* Swift 中的协议
  * 从一个绘图应用开始。通过实现一个绘图应用，来讲解在 Swift 中使用协议
  * 带有 Self 和关联类型的协议
    * 带有 Self 的协议。通过实现一个二分查找，来讲解如何在协议中使用 Self
    * 带有关联类型的协议。通过实现一个带加载动画的数据加载器，来讲解如何在协议中使用关联类型
  * 协议与函数派发。通过一个使用多态的例子，来讲解函数派发在协议中的表现
* 使用协议改善既有的代码设计



## 面向协议编程不是个新概念

面向协议编程并不是一个新概念，它其实就是广为所知的面向接口编程。面向协议编程 (Protocol Oriented Programming) 是 Apple 在 2015 年 WWDC 上提出的 Swift 的一种编程范式。



很多程序员都能理解类、对象、继承和接口这些面向对象的概念（不知道的自己面壁去啊）。可是类与接口的区别何在？有类了干嘛要使用接口？相信很多人都有这样的疑问。接口（协议是同一个东西）定义了类型，实现接口（子类型化）让我们可以用一个对象来代替另一个对象。另一方面，类继承是通过复用父类的功能或者只是简单地共享代码和表述，来定义对象的实现和类型的一种机制。类继承让我们能够从现成的类继承所需要大部分功能，从而快速定义新的类。所以接口侧重的是类型（是把某个类型当做另一种类型来用），而类侧重的是复用。理解了这个区别你就知道在什么时候使用接口，什么时候使用类了。

GoF 在《设计模式》一书中提到了可复用面向对象软件设计的原则：

> 针对接口编程，而不是针对实现编程

定义具有相同接口的类群很重要，因为多态是基于接口的。其他面向对象的编程语言，类如 Java，允许我们定义 "接口"类型，它确定了客户端同所有其他具体类直接到一种 "合约"。Objective-C 和 Swift中与之对应的就是协议（protocol）了。协议也是对象之间的一种合约，但本身是不能够实例化为对象的。实现协议或者从抽象类继承，使得对象共享相同的接口。因此，子类型的所有对象，都可以针对协议或抽象类的接口做出应答。



## Swift 中的协议

在 WWDC2015 上，Apple 发布了Swift 2。新版本包含了很多新的语言特性。在众多改动之中，最引人注意的就是 protocol extensions。在 Swift 第一版中，我们可以通过 extension 来为已有的 class，struct 或 enum 拓展功能。而在 Swift 2 中，我们也可以为 protocol 添加 extension。可能一开始看上去这个新特性并不起眼，实际上 protocol extensions 非常强大，以至于可以改变 Swift 之前的某些编程思想。后面我会给出一个 protocol extension 在实际项目中使用案例。



除了协议拓展，Swift 中的协议还有一些具有其他特性的协议，比如带有关联类型的协议、包含 Self 的协议。这两种协议跟普通的协议还是有一些不同的，后面我也会给出具体的例子。



我们现在可以开始编写代码，来掌握在实际开发中使用 Swift 协议的技巧。下面的绘图应用和二分查找的例子是来自 WWDC2015 中[这个 Session](https://developer.apple.com/videos/play/wwdc2015/408/)。在写本文前，笔者也想了很多例子，但是始终觉得没有官方的例子好。所以我的建议是：[这个 Session](https://developer.apple.com/videos/play/wwdc2015/408/) 至少要看一遍。看了一遍后，开始写自己的实现。



### 从一个绘图应用开始

现在我们可以先通过完成一个具体的需求，来学习如何在 Swift 中使用协议。

我们的需求是实现一个可以绘制复杂图形的绘图程序，我们可以先通过一个 Render 来定义一个简单的绘制过程：

```swift
struct Renderer {
    func move(to p: CGPoint) { print("move to (\(p.x), \(p.y))") }
    
    func line(to p: CGPoint) { print("line to (\(p.x), \(p.y))")}
    
    func arc(at center: CGPoint, radius: CGFloat, starAngle: CGFloat, endAngle: CGFloat) {
        print("arc at center: \(center), radius: \(radius), startAngel: \(starAngle), endAngle: \(endAngle)")
    }
}
```

然后可以定义一个 Drawable 协议来定义一个绘制操作：

```swift
protocol Drawable {
    func draw(with render: Renderer)
}
```

Drawable 协议定义了一个绘制操作，它接受一个具体的绘制工具来进行绘图。这里将可绘制的内容和实际的绘制操作分开了，这么做的目的是为了职责分离，在后面你会看到这种设计的好处。



如果我们想绘制一个圆，我们可以很简单地利用上面实现好了的绘制工具来绘制一个圆形，就像下面这样：

```swift
struct Circle: Drawable {
    let center: CGPoint
    let radius: CGFloat
    
    func draw(with render: Renderer) {
        render.arc(at: center, radius: radius, starAngle: 0, endAngle: CGFloat.pi * 2)
    }
}
```

现在我们又想要绘制一个多边形，那么有了 Drawable 协议，实现起来也非常简单：

```swift
struct Polygon: Drawable {
    let corners: [CGPoint]
    
    func draw(with render: Renderer) {
        if corners.isEmpty { return }
        render.move(to: corners.last!)
        for p in corners { render.line(to: p) }
    }
}
```

简单图形的绘制已经完成了，现在可以完成我们这个绘图程序了：

```swift
struct Diagram: Drawable {
    let elements: [Drawable]
    
    func draw(with render: Renderer) {
        for ele in elements { ele.draw(with: render) }
    }
}

let render = Renderer()

let circle = Circle(center: CGPoint(x: 100, y: 100), radius: 100)
let triangle = Polygon(corners: [
    CGPoint(x: 100, y: 0),
    CGPoint(x: 0, y: 150),
    CGPoint(x: 200, y: 150)])

let client = Diagram(elements: [triangle, circle])
client.draw(with: render)

// Result:
// move to (200.0, 150.0)
// line to (100.0, 0.0)
// line to (0.0, 150.0)
// line to (200.0, 150.0)
// arc at center: (100.0, 100.0), radius: 100.0, startAngel: 0.0, endAngle: 6.28318530717959
```

通过上面的代码很容易就实现了一个简单的绘图程序了。不过，目前这个绘图程序只能在控制台中显示绘制的过程，我们想把它绘制到屏幕上怎么办呢？要想把内容绘制到屏幕上其实也简单的很，仍然是使用协议，我们可以把 Renderer 结构体改成 protocol：

```swift
protocol Renderer {
    func move(to p: CGPoint)
    
    func line(to p: CGPoint)
    
    func arc(at center: CGPoint, radius: CGFloat, starAngle: CGFloat, endAngle: CGFloat)
}
```

完成了 Renderer 的改造，我们可以使用 CoreGraphics 来在屏幕上绘制图形了：

```swift
extension CGContext: Renderer {
    func line(to p: CGPoint) {
        addLine(to: p)
    }
    
    func arc(at center: CGPoint, radius: CGFloat, starAngle: CGFloat, endAngle: CGFloat) {
        let path = CGMutablePath()
        path.addArc(center: center, radius: radius, startAngle: starAngle, endAngle: endAngle, clockwise: true)
        addPath(path)
    }
}
```

通过拓展 CGContext，使其遵守 Renderer 协议，然后使用 CGContext 提供的接口非常简单的实现了绘制工作。 下图是这个绘图程序最终的效果：

![](render.png)

完成上面绘图程序的关键，是将图形的定义和实际绘制操作拆开了，通过设计 `Drawable` 和 `Renderer` 两个协议，完成了一个高拓展的程序。想绘制其他形状，只要实现一个新的 Drawable 就可以了。例如我想绘制下面这样的图形：

<img src="render2.png" width="390px" height="390px" />

我们可以将原来的 Diagram 进行缩放就可以了。代码如下：

```swift
let big = Diagram(elements: [triangle, circle])
diagram = Diagram(elements: [big, big.scaled(by: 0.2)])
```

而通过实现 Renderer 协议，你既可以完成基于控制台的绘图程序也可以完成使用 CoreGraphics 的绘图程序，甚至可以很简单地就能实现一个使用 OpenGL 的绘图程序。这种编程思想，在编写跨平台的程序是非常有用的。



### 带有 Self 和关联类型的协议

我在前面部分已经指出，带有关联类型的协议和普通的协议是有一些不同的。对于那些在协议中使用了 Self 关键字的协议来说也是如此。在 Swift 3 中，这样的协议不能被当作独立的类型来使用。这个限制可能会在今后实现了完整的泛型系统后被移除，但是在那之前，我们都必须要面对和处理这个限制。



#### 带有 Self 的协议

我们仍然从一个例子开始：

```swift
func binarySearch(_ keys: [Int], for key: Int) -> Int {
    var lo = 0, hi = keys.count - 1
    while lo <= hi {
        let mid = lo + (hi - lo) / 2
        if keys[mid] == key { return mid }
        else if keys[mid] < key { lo = mid + 1 }
        else { hi = mid - 1 }
    }
    return -1
}

let position = binarySearch([Int](1...10), for: 3)
// result: 2
```

上面的代码实现了一个简单的二分查找，但是目前只支持查找 Int 类型的数据。如果想支持其他类型的数据，我们必须对上面的代码进行改造，改造的方向就是使用 protocol，例如我可以添加下面的实现：

```swift
protocol Ordered {
    func precedes(other: Ordered) -> Bool
    
    func equal(to other: Ordered) -> Bool
}

func binarySearch(_ keys: [Ordered], for key: Ordered) -> Int {
    var lo = 0, hi = keys.count - 1
    while lo <= hi {
        let mid = lo + (hi - lo) / 2
        if keys[mid].equal(to: key) { return mid }
        else if keys[mid].precedes(other: key) { lo = mid + 1 }
        else { hi = mid - 1 }
    }
    return -1
}
```

为了支持查找 Int 类型数据，我们就必须让 Int 实现 `Oredered` 协议：

![](binarysearch1.png)

写完上面的实现，发现代码根本就不能执行，报错说的是 Int 类型和 Oredered 类型不能使用 `<` 进行比较，下面的 `==` 也是一样。为了解决这个问题，我们可以在 protocol 中使用 Self：

```swift
protocol Ordered {
    func precedes(other: Self) -> Bool
    
    func equal(to other: Self) -> Bool
}

extension Int: Ordered {
    func precedes(other: Int) -> Bool { return self < other }
    
    func equal(to other: Int) -> Bool { return self == other }
}
```

在 Oredered 中使用了 Self 后，编译器会在实现中将 Self 替换成具体的类型，就像上面的代码中，将 Self 替换成了 Int。这样我们就解决了上面的问题。但是又出现了新的问题：

![](binarysearch2.png)

这就是上面所说的，带有 Self 的协议不能被当作独立的类型来使用。在这种情况下，我们可以使用泛型来解决这个问题：

```swift
func binarySearch<T: Ordered>(_ keys: [T], for key: T) -> Int {...}
```

如果是 String 类型的数据，也可以使用这个版本的二分查找了：

```swift
extension String: Ordered {
    func precedes(other: String) -> Bool { return self < other }
    
    func equal(to other: String) -> Bool { return self == other }
}

let position = binarySearch(["a", "b", "c", "d"], for: "d")
// result: 3
```

当然，如果你熟悉 Swift 标准库中的协议的话，你会发现上面的实现可以简化为下面的几行代码：

```swift
func binarySearch<T: Comparable>(_ keys: [T], for key: T) -> Int? {
    var lo = 0, hi = keys.count - 1
    while lo <= hi {
        let mid = lo + (hi - lo) / 2
        if keys[mid] == key { return mid }
        else if keys[mid] < key { lo = mid + 1 }
        else { hi = mid - 1 }
    }
    return nil
}
```

这里我们定义 Ordered 协议只是为了演示在协议中使用 Self 的过程。实际开发中，可以灵活地运用标准库中提供的协议。其实在标准库中 Comparable 协议中也是用到了 Self 的：

```swift
extension Comparable {
    public static func > (lhs: Self, rhs: Self) -> Bool
}
```



上面通过实现一个二分查找算法，演示了如何使用带有 Self 的协议。简单来讲，你可以把 Self 看做一个占位符，在后面具体类型的实现中可以替换成实际的类型。



#### 带有关联类型的协议

带有关联类型的协议也不能被当作独立的类型来使用。在 Swift 中这样的协议非常多，例如 Collection，Sequence，IteratorProtocol 等等。如果你仍然想使用这种协议作为类型，可以使用一种叫做类型擦除的技术。你可以从[这里](http://jewelz.me/cjs77iamr00028is6rdrwj0cw/)了解如何实现它。

下面仍然通过一个例子来演示如何在项目中使用带有关联类型的协议。这次我们要通过协议实现一个带有加载动画的数据加载器，并且在出错时展示相应的占位图。

这里，我们定义了一个 Loading 协议，代表可以加载数据，不过要满足 Loading 协议，必须要提供一个 `loadingView`，这里的 `loadingView` 就是协议中关联类型的实例。

```swift
protocol Loading: class {
    associatedtype LoadingView: UIView, LoadingViewType
    
    var loadingView: LoadingView { get }
}
```

 Loading 协议中的关联类型有两个要求，首先必须是 UIView 的子类，其次需要遵守 LoadingViewType 协议。LoadingViewType 可以简单定义成下面这样：

```swift
protocol LoadingViewType: class {
    var isAnimating: Bool { get set }
    var isError: Bool { get set }
    
    func startAnimating()
    func stopAnimating()
}
```

我们可以在 Loading 协议的拓展中定义一些跟加载逻辑相关的方法：

```swift
extension Loading where Self: UIViewController {
    func startLoading() {
        if !view.subviews.contains(loadingView) {
            view.addSubview(loadingView)
            loadingView.frame = view.bounds
        }
        view.bringSubview(toFront: loadingView)
        loadingView.startAnimating()
    }
    
    func stopLoading() {
        loadingView.stopAnimating()
    }
}

```

我们可以继续给 Loading 添加一个带网络数据加载的逻辑：

```swift
extension Loading where Self: UIViewController {
    func loadData(with re: Resource, completion: @escaping (Result) -> Void) {
        startLoading()
        NetworkTool.shared.request(re) { result in
            guard case .succeed = result else {
                self.loadingView.isError = true // 显示出错的视图，这里可以根据错误类型显示对应的视图，这里简单处理了
                self.stopLoading()
                return
            }
            completion(result)
            self.loadingView.isError = false
            self.stopLoading()
        }
    }
}
```

以上就是整个 Loading 协议的实现。这里跟上面的例子不同，这儿主要使用了协议的拓展来实现需求。这样做的原因，是因为所有的加载逻辑几乎都是一样的，可能的区别就是加载的动画不同。所以这里把负责动画的部分放到了  LoadingViewType 协议里，Loading 的加载逻辑都放到协议的拓展里进行定义。协议声明里定义的方法与在协议拓展里定义的方法其实是有区别的，后面也会给出一个例子来说明它们都区别。

要想让 ViewController 有加载数据的功能，只要让控制器遵守 Loading 协议就行，然后在合适的地方调用 `loadData` 方法：

```swift
class ViewController: UIViewController, Loading {
    var loadingView = TestLoadingView()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        loadData(with: Test.justEmpty) { print($0) }
    }
}
```

下面是运行结果：

![](loading.gif)

我们只要让控制器遵守 Loading 协议，就实现了从网络加载数据并带有加载动画，而且在出错时显示错误视图的功能。这里肯定有人会说，使用继承也可以实现上述需求。当然，我们可以把协议中的加载逻辑都放到一个基类中，也可以实现该需求。如果后面又要添加刷新和分页功能，那么这些代码也只能放到基类中，这样就会随着项目越来越大，基类也变得越来越臃肿，这就是所谓的上帝类。如果我们将数据加载、刷新、分页作为不同的协议，让控制器需要什么就遵守相应的协议，那么控制器就不会包含那些它不需要的功能了。这就像搭积木一样，可以灵活地给程序添加它需要的内容。



### 协议与函数派发

函数派发就是一个程序在调用一个方法时，如何选择要执行的指令的过程。当我们每次调用一个方法时函数派发都会发生。

编译型语言有三种基础的函数派发方式：直接派发(Direct Dispatch)，函数表(Table Dispatch) 和消息(Message Dispatch)。大部分语言支持一到两种。Java 默认使用函数表派发，你可以通过使用 `final` 关键字将其变为直接派发。C++ 默认使用直接派发，通过 `virtual` 关键字可以改为函数表派发。Objective-C 总是使用消息派发，但允许开发者使用 C 直接派发来获取性能的提高（比如直接调用 IMP）。Swift 在这方面走在了前面，她支持全部的3种派发方式。这样的方式非常好,，不过也给很多Swift开发者带来了困扰。

这里只简单说一下函数派发在 protocol 中的不同表现。看下面的例子：

```swift
protocol Flyable {
    func fly()
}
```

上面定义了 Flyable 协议，表示了飞行的能力。遵守该协议就必须实现 `fly()` 方法。我们可以提供几个实现：

```swift
struct Eagle: Flyable {
    func fly() { print("🦅 is flying") }
}

struct Plane: Flyable {
    func fly() { print("✈️ is flying") }
}
```

写个客户端程序测试一下：

```swift
let fls: [Flyable] = [Eagle(), Plane()]
for fl in fls {
    fl.fly()
}

// result:
🦅 is flying
✈️ is flying
```

上面测试程序的运行结果和我们的设想完全一样。上面 `fly()` 方法是在协议的定义里进行声明的，现在我们把它放到协议拓展里进行声明，就像下面这样：

```swift
extension Flyable {
    func fly() { print("Something is flying") }
}
```

在运行前你可以先猜测一下运行的结果。

先暂停 3 秒钟...



下面是运行结果：

```
Something is flying
Something is flying
```

你看，我们只是简单地把在协议定义里的方法挪到了协议拓展里，运行结果却完全不同。出现像上面那样的运行结果还跟这行代码有关：

```swift
let fls: [Flyable] = [Eagle(), Plane()]
```

如果你直接使用具体类型进行调用，肯定是没有问题的，就像下面这样：

```swift
Eagle().fly() 	// 🦅 is flying
Plane().fly() 	// ✈️ is flying
```

出现上面两种完全不同的结果，主要是因为函数派发根据方法声明的位置的不同而采用了不同的策略，总结起来有这么几点:

- 值类型（struct, enum）总是会使用直接派发
- 而协议和类的 `extension` 都会使用直接派发
- 协议和普通 Swift 类声明作用域里的方法都会使用函数表进行派发
- 继承 `NSObject` 的类声明作用域里的方法都会使用函数表派发
- 继承 `NSObject` 的类的 `extension` 、使用 `dynamic` 标记的方法会使用消息派发

下面这张图很清楚地总结了 Swift 中函数派发方式，不过少了 `dynamic` 的方式。

![](https://www.raizlabs.com/dev/wp-content/uploads/sites/10/2016/12/Defaults-1-768x503.png)

在上面的例子中，虽然 `Eagle` 和 `Plane` 都实现了 `fly()` 方法，但在多态时，仍然会调用协议拓展里的默认实现。因为，在协议拓展声明的方法，在调用时，使用的是直接派发，直接派发总是要优于其他的派发方式的。

所以理解 Swift 中的函数派发，对于我们写出结构清晰、没有 bug 的代码是非常重要的。当然，如果你没有使用到多态，直接使用具体的类型，是不会出现上面的问题的。既然你都开始 "针对接口编程，而不是针对实现编程"，怎么会用不到多态呢，是吧。



## 使用协议改善既有的代码设计

通过上面的例子可以看出，通过协议进行代码共享相比与通过继承的共享，有这几个优势：

- 我们不需要被强制使用某个父类。
- 我们可以让已经存在的类型满足协议 (比如我们让 CGContext 满足了 Renderer)。子类就没那么灵活了，如果 CGContext 是一个类的话，我们无法以追溯的方式去变更它的父类。
- 协议既可以用于类，也可以用于结构体、枚举，而继承就无法和结构体、枚举一起使用了。
- 协议可以模拟多继承。
- 最后，当处理协议时，我们无需担心方法重写或者在正确的时间调用 super 这样的问题。



通过面向协议的编程，我们可以从传统的继承上解放出来，用一种更灵活的方式，像搭积木一样对程序进行组装。协议和类一样，在设计时要遵守 "单一职责" 原则，让每个协议专注于自己的功能。得益于协议扩展，我们可以减少继承带来的共享状态的风险，让代码更加清晰。

使用面向协议编程有助于我们写出低耦合、易于扩展以及可测试的代码，而结合泛型来使用协议，更可以让我们免于动态调用和类型转换的苦恼，保证了代码的安全性。

