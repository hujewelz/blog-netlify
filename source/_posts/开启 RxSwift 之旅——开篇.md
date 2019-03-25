---
title: 开启 RxSwift 之旅——开篇
date: 2017-06-01 10:08:13
tags: 
  - Swift 
  - 响应式编程
category: Swift
thumbnailImage: rxlogo.png
---

RxSwift 是 ReactiveX 在 Swift 下的实现。ReactiveX 是一个通过使用可观察序列来组合异步和基于事件的程序的库。

<!--more-->

很多地方通常把 ReactiveX 称为 “函数响应式编程” ，其实这是不恰当的。ReactiveX 可以是函数式的，可以是响应式的，但是和“函数响应式编程”是不同的概览。一个主要的不同点是“函数响应式编程”是对随着时间不停变化的值进行操作的，而 ReactiveX 对随时间发射的离散值进行操作。

我们先不急着去看 RxSwift 的源码，在这之前，我们有必要先了解一下什么是响应式编程。



## "什么是响应式编程"

> 响应式编程是一种面向数据流和变化传播的编程范式。

在某种程度上，这并不是什么新东西。用户输入、单击事件、变量值等都可以看做一个流，你可以观察这个流，并基于这个流做一些操作。响应式就是基于这种想法。

一个流就是一个将要发生的以时间为序的事件序列。它能发射出三种不同的东西：一个数据值(某种类型的)，一个错误（error）或者一个“完成（completed）”的信号。比如说，当前按钮所在的窗口或视图关闭时，“单击”事件流也就“完成”了。

以一个单击事件流为例：定义一个针对数据值的函数，在发出一个值时，该函数就会异步地执行，还有一个针对发出错误时的函数，最后还有针对发出‘完成’时的函数。“监听”流的行为叫做订阅。我们定义的这些函数就是观察者。这个流就是被观察的主体(subject)（或“可观察的(observable)”）。这正是观察者设计模式。

在你使用 RxSwift 时，你就会发现它正是按照这种模式来进行设计的。在 RxSwift 中，一个流可以被称为序列(Sequences)。序列的生产者就是 Observable 。

在 RxSwift 的 playground 中就有这么一句话：
> Every Observable instance is just a sequence.

## Observable

如果你在学习 RxSwift 之前就使用过 ReactiveCocoa 的话，你会发现 RxSwift 和 ReactiveCocoa 完全是两个不同的物种。在 RxSwift 的世界里，所有的东西都是 Observable 的。你可以创造它们、操作它们，然后订阅它们来响应变化。

理解 Observable 还有一件很重要的事情：
> Observables will not execute their subscription closure unless there is a subscriber. 

可以这么理解，如果你只是调用一个返回一个 Observable 的方法，生成序列不会被执行。Observable 只是一个解释序列如何被生成和什么参数被使用于生成元素的定义。生成序列开始于 subscribe 方法被调用的时候。

下面的例子中，Observable 的闭包永远不会执行：

```swift
example("Observable with no subscribers") {
    _ = Observable<String>.create { observer -> Disposable in
        print("This will never be printed")
        observer.on(.next("😬"))
        observer.on(.completed)
        return Disposables.create()
    }
}
```

只有当我们调用 `subscribe(_:)` 时，Observable 的闭包才会执行：

```swift
example("Observable with subscriber") {
  _ = Observable<String>.create { observer in
            print("Observable created")
            observer.on(.next("😉"))
            observer.on(.completed)
            return Disposables.create()
        }
        .subscribe { event in
            print(event)
    }
}
```

上面例子中从传入闭包创建一个 Observable ，到调用 `subscribe(_:)` 这个过程中 RxSwift 到底做了什么？我们可以先从简单的 empty 开始。

### empty

empty 就是创建一个空的 sequence, 它只能发出一个 completed 事件。

```swift
example(of: "empty") {
    Observable<Int>.empty()
        .subscribe({
            print($0)
    }) 
}

// 打印结果
--- Example of: empty ---
completed

```

上面代码中通过 Observable 的 `empty` 方法创建了一个 `Observable<Int>`, 打开 Observable+Creation.swift 文件，可以看到 `empty()` 的实现：

```swift
public static func empty() -> Observable<E> {
    return EmptyProducer<E>()
}
```

这里返回了一个 `EmptyProducer` 的实例，点进去看看`EmptyProducer`是个什么东西：

```swift
final class EmptyProducer<Element> : Producer<Element> {
    override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
        observer.on(.completed)
        return Disposables.create()
    }
}
```

`EmptyProducer`是 `Producer` 的子类，重写了 `subscribe(:)` 。在 subscribe 方法中，观察者订阅了一个完成信号。

当我们通过 `empty()` 创建了一个 Observable 后，然后会调用 `subscribe(_:)`，打开 ObservableType+Extensions.swift 文件, 可以看到 subscribe 方法的实现：

```swift
public func subscribe(_ on: @escaping (Event<E>) -> Void) -> Disposable {
    let observer = AnonymousObserver { e in
        on(e)
    }
    return self.subscribeSafe(observer)
}
```
subscribe 方法接受了闭包之后，先创建了一个匿名观察者，subscribe 的闭包参数作为构造器的参数传给了 observer。点击进去 AnonymousObserver源码：

```swift
final class AnonymousObserver<ElementType> : ObserverBase<ElementType> {
    typealias Element = ElementType
    
    typealias EventHandler = (Event<Element>) -> Void
    
    private let _eventHandler : EventHandler
    
    init(_ eventHandler: @escaping EventHandler) {
#if TRACE_RESOURCES
        let _ = Resources.incrementTotal()
#endif
        _eventHandler = eventHandler
    }

    override func onCore(_ event: Event<Element>) {
        return _eventHandler(event)
    }
    
#if TRACE_RESOURCES
    deinit {
        let _ = Resources.decrementTotal()
    }
#endif
}
```

AnonymousObserver 的构造器接受一个闭包，然后在 onCore 方法中， 私有的 `_eventHandler` 会被调用。到这里为止，我们还是不知道我们在调用 `subscribe(_:)` 时传入的闭包最终的调用时机。不过已经很清楚的知道了，这个闭包在 `onCore(:)` 中调用了，我们继续进入 AnonymousObserver 的父类 ObserverBase 中一探究竟：

```swift
class ObserverBase<ElementType> : Disposable, ObserverType {
    typealias E = ElementType

    private var _isStopped: AtomicInt = 0

    func on(_ event: Event<E>) {
        switch event {
        case .next:
            if _isStopped == 0 {
                onCore(event)
            }
        case .error, .completed:
            if AtomicCompareAndSwap(0, 1, &_isStopped) {
                onCore(event)
            }
        }
    }

    func onCore(_ event: Event<E>) {
        rxAbstractMethod()
    }

    func dispose() {
        _ = AtomicCompareAndSwap(0, 1, &_isStopped)
    }
}
```
这一下就很清楚了，`onCore(:)` 会被 `on(:)` 调用。让我们再次回到 ObservableType+Extensions.swift 文件中，匿名观察者(AnonymousObserver)创建完后，调用 `subscribeSafe(:)` 作为函数返回值。在文件的最下面可以看到 `subscribeSafe(:)` 的实现：

```swift
fileprivate func subscribeSafe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E {
    return self.asObservable().subscribe(observer)
}
```

这里会调用 `subscribe(:)` ，注意了，这里的 `subscribe(:)` 是 ObservableType 协议中定义的方法：

```swift 
public protocol ObservableType : ObservableConvertibleType {
    
    associatedtype E
    
    func subscribe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E
}
```

这里的参数是一个 ObserverType，也就是一个观察者，千万要与 `func subscribe(_ on: @escaping (Event<E>) -> Void) -> Disposable` 做好区分。

好了， subscribe 方法将创建的匿名观察者作为参数，而在 EmptyProducer 中的 subscribe 的实现我们已经看过了：

```
override func subscribe<O : ObserverType>(_ observer: O) -> Disposable where O.E == Element {
    observer.on(.completed)
    return Disposables.create()
}
```

这里刚好调用了观察者的 `on(:)`, 在 ObserverBase 中 on 方法会调用 `onCore(:)`, onCore 方法调用了 `subscribe(_ on: @escaping (Event<E>) -> Void) -> Disposable` 参数中的闭包。由于 `subscribe(_ observer: O)` 中观察者只订阅了 "completed" 信号，所有闭包不会执行。

至此从创建一个 observable， 到调用 `subscribe(_:)` 整个过程我们已经很清楚了。现在也就能明白为什么只是调用一个返回一个 Observable 的方法，生成序列不会被执行了。

## 小结
最后总结一下调用 `subscribe(_:)` 后的整个过程：用 subscribe 中的闭包创建一个匿名观察者（观察者私有的 `_eventHandler` 会将闭包保存起来），然后将创建的匿名观察者作为参数传给 `subscribeSafe(:)` , `subscribeSafe(:)` 会调用 `subscribe(:)`, 并将匿名观察者作为参数。`subscribe(:)` 会调用 observer 的 `on(:)`, 当 observer 的 on 方法被调用后，最终会调用开始时传入的闭包。

以上只是分析了一下 empty 的实现，像 of, just, create 等的实现在细节上有一些区别，总的思路是一样的。在查看源码时可能会有一点绕，主要是因为继承太多，很多方法都要到父类中去找，而且 ObservableType 和 ObserverType 的 Extension 太多，代码分散到各个文件中。

RxSwift 的代码只看了个开头，还有很多地方没有完全弄明白。在使用 RxSwift 的过程中你能体会到 "响应式" 和 "函数式" 给我们的开发带来的便利性。

