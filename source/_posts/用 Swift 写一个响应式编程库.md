---
title: 用 Swift 写一个响应式编程库
date: 2017-12-02 17:18:21
tags:
  - Swift
  - Reactive programing
categories: Swift
thumbnailImage: cover.png
coverImage: cover.png
---

2017年又快过去了，忙了一年感觉没啥收获，感觉是不是应该写点啥，想了好久没想出要写什么。下半年因为工作的原因，狗狗也没养了，吉他上也积满了灰尘，

<!--more-->

兴致勃勃的学习素描，到现在也没画出了啥😂，博客也很久没更新了。想想感觉更新一下博客吧。

整个2017年我完全使用 Swift 进行开发了。使用 Swift 进行开发是一个很愉快的体验，我已经完全不想再去碰 OC 了。最近想做一个响应式编程的库，所以就把它拿来分享一下。


## Reactive Programing 

说到响应式编程，ReactiveCocoa 和 RxSwift 可以说是目前 iOS 开发中最优秀的第三方开源库了。今天咱们不聊 ReactiveCocoa 和 RxSwif，咱们自己来写一个响应式编程库。如果你对观察者模式很熟悉的话，那么响应式编程就很容易理解了。

> 响应式编程是一种面向数据流和变化传播的编程范式。

比如用户输入、单击事件、变量值等都可以看做一个流，你可以观察这个流，并基于这个流做一些操作。“监听”流的行为叫做订阅。响应式就是基于这种想法。

 废话不多说，撸起袖子开干。

我们以一个获取用户信息的网络请求为例：

```swift
func fetchUser(with id: Int, completion: @escaping ((User) -> Void)) {
     DispatchQueue.main.asyncAfter(deadline: DispatchTime.now()+2) {
         let user = User(name: "jewelz")
         completion(user)
     }
}
```

上面是我们通常的做法，在请求方法里传入一个回调函数，在回调里拿到结果。在响应式里面，我们监听请求，当请求完成时，观察者得到更新。

```swift
func fetchUser(with id: Int) -> Signal<User> {}
```

发送网络请求就可以这样：

```swift
fetchUser(with: "12345").subscribe({
    
})
```

在完成 Signal 之前， 需要定义订阅后返回的数据结构，这里我只关心成功和失败两种状态的数据，所以可以这样写:

```swift
enum Result<Value> {
    case success(Value)
    case error(Error)
}
```

现在可以开始实现我们的 Signal 了:

```swift
final class Signal<Value> {
    fileprivate typealias Subscriber = (Result<Value>) -> Void
  	fileprivate var subscribers: [Subscriber] = []
  
    func send(_ result: Result<Value>) {
        for subscriber in subscribers {
            subscriber(result)
        }
    }
  
    func subscribe(_ subscriber: @escaping (Result<Value>) -> Void) {
        subscribers.append(subscriber)
    }
}
```

写个小例子测试一下：

```swift
let signal = Signal<Int>()
signal.subscribe { result in
    print(result)
}
signal.send(.success(100))
signal.send(.success(200))

// Print
success(100)
success(200)
```

我们的 Signal 已经可以正常工作了，不过还有很多改进的空间，我们可以使用一个工厂方法来创建一个 Signal, 同时将 `send `变为私有的：

```swift
static func empty() -> ((Result<Value>) -> Void, Signal<Value>) {
     let signal = Signal<Value>()
     return (signal.send, signal)
}

fileprivate func send(_ result: Result<Value>) { ... }
```

现在我们需要这样使用 Signal 了：

```swift
let (sink, signal) = Signal<Int>.empty()
signal.subscribe { result in
    print(result)
}
sink(.success(100))
sink(.success(200))
```

接着我们可以给 UITextField 绑定一个 Signal，只需要在 Extension 中给 UITextField添加一个计算属性  ：

```
extension UITextField {
    var signal: Signal<String> {
        let (sink, signal) = Signal<String>.empty()
        let observer = KeyValueObserver<String>(object: self, keyPath: #keyPath(text)) { str in
            sink(.success(str))
        }
        signal.objects.append(observer)
        return signal
    }
}
```

上面代码中的 `observer` 是一个局部变量，在 `signal`调用完后，就会被销毁，所以需要在 Signal 中保存该对象，可以给 Signal 添加一个数组，用来保存需要延长生命周期的对象。 KeyValueObserver 是对 KVO 的简单封装，其实现如下：

```swift
final class KeyValueObserver<T>: NSObject {
    
    private let object: NSObject
    private let keyPath: String
    private let callback: (T) -> Void
    
    init(object: NSObject, keyPath: String, callback: @escaping (T) -> Void) {
        self.object = object
        self.keyPath = keyPath
        self.callback = callback
        super.init()
        object.addObserver(self, forKeyPath: keyPath, options: [.new], context: nil)
    }
    
    override func observeValue(forKeyPath keyPath: String?, of object: Any?, change: [NSKeyValueChangeKey : Any]?, context: UnsafeMutableRawPointer?) {
        guard let keyPath = keyPath, keyPath == self.keyPath, let value = change?[.newKey] as? T else { return }
      
        callback(value)
    }
    
    deinit {
        object.removeObserver(self, forKeyPath: keyPath)
    }
}
```

现在就可以使用` textField.signal.subscribe({})` 来观察 UITextField 内容的改变了。

 在 Playground 写个 VC 测试一下：

```swift
class VC {
    let textField =  UITextField()
    var signal: Signal<String>?
    
    func viewDidLoad() {
        signal = textField.signal
        signal?.subscribe({ result in
            print(result)
        })
        textField.text = "1234567"
    }
    
    deinit {
        print("Removing vc")
    }
}

var vc: VC? = VC()
vc?.viewDidLoad()
vc = nil

// Print
success("1234567")
Removing vc
```



## Reference Cycles

我在上面的 Signal 中，添加了 `deinit`方法：

```swift
deinit {
	print("Removing Signal")
}

```

最后发现 Signal 的析构方法并没有执行，也就是说上面的代码中出现了循环引用，其实仔细分析上面 UITextField 的拓展中 `signal`的实现就能发现问题出在哪儿了。

```swift
let observer = KeyValueObserver<String>(object: self, keyPath: #keyPath(text)) { str in
    sink(.success(str))
}
```

在 `KeyValueObserver` 的回调中，调用了 `sink()`方法，而 `sink` 方法其实就是 `signal.send(_:)`方法，这里在闭包中捕获了` signal` 变量，于是就形成了循环引用。这里只要使用 `weak` 就能解决。修改下的代码是这样的：

```swift
static func empty() -> ((Result<Value>) -> Void, Signal<Value>) {
     let signal = Signal<Value>()
     return ({[weak signal] value in signal?.send(value)}, signal)
}
```

再次运行， Signal 的析构方法就能执行了。

上面就实现了一个简单的响应式编程的库了。不过这里还存在很多问题，比如我们应该在适当的时机移除观察者，现在我们的观察者被添加在 ` subscribers` 数组中，这样就不知道该移除哪一个观察者，所以我们将数字替换成字典，用 UUID   作为 key :

```swift 
fileprivate typealias Token = UUID
fileprivate var subscribers: [Token: Subscriber] = [:]
```

我们可以模仿 RxSwift 中的 Disposable 用来移除观察者，实现代码如下：

```swift
final class Disposable {
    private let dispose: () -> Void
    
    static func create(_ dispose: @escaping () -> Void) -> Disposable {
        return Disposable(dispose)
    }
    
    init(_ dispose: @escaping () -> Void) {
        self.dispose = dispose
    }
    
    deinit {
        dispose()
    }
}
```

原来的 `subscribe(_:)` 返回一个 Disposable 就可以了:

```swift
func subscribe(_ subscriber: @escaping (Result<Value>) -> Void) -> Disposable {
     let token = UUID()
     subscribers[token] = subscriber
      return Disposable.create {
          self.subscribers[token] = nil
      }   
 }
```

这样我们只要在适当的时机销毁 Disposable 就可以移除观察者了。

作为一个响应式编程库都会有 `map`, `flatMap`, `filter`, `reduce` 等方法，所以我们的库也不能少，我们可以简单的实现几个。



## map

map 比较简单，就是将一个 *返回值为包装值的函数* 作用于一个**包装(Wrapped)值**的过程， 这里的包装值可以理解为可以包含其他值的一种结构，例如 Swift 中的数组，可选类型都是包装值。它们都有重载的 `map`, `flatMap`等函数。以数组为例，我们经常这样使用：

```swift
let images = ["1", "2", "3"].map{ UIImage(named: $0) }
```

现在来实现我们的 map 函数：

```swift
func map<T>(_ transform: @escaping (Value) -> T) -> Signal<T> {
     let (sink, signal) = Signal<T>.empty()
     let dispose = subscribe { (result) in
          sink(result.map(transform))
      }
      signal.objects.append(dispose)
      return signal
}
```

我同时给 Result 也实现了 map 函数:

```swift
extension Result {
    func map<T>(_ transform: @escaping (Value) -> T) -> Result<T> {
        switch self {
        case .success(let value):
            return .success(transform(value))
        case .error(let error):
            return .error(error)
        }
    }
}

// Test

let (sink, intSignal) = Signal<Int>.empty()
intSignal
    .map{ String($0)}
    .subscribe {  result in
        print(result)
}
sink(.success(100))

// Print success("100")
```



## flatMap

flatMap 和 map 很相似，但也有一些不同，以可选型为例，Swif t是这样定义 map 和 flatMap 的：

```swift
public func map<U>(_ transform: (Wrapped) throws -> U) rethrows -> U?
public func flatMap<U>(_ transform: (Wrapped) throws -> U?) rethrows -> U?
```

 flatMap 和 map 的不同主要体现在 transform 函数的返回值不同。map 接受的函数返回值类型是 `U`类型，而 flatMap 接受的函数返回值类型是 ` U?`类型。例如对于一个可选值，可以这样调用：

```swift
let aString: String? = "￥99.9"
let price = aString.flatMap{ Float($0)}

// Price is nil
```

我们这里 flatMap 和 Swift 中数组以及可选型中的 flatMap 保持了一致。

所以我们的 flatMap 应该是这样定义：`flatMap<T>(_ transform: @escaping (Value) -> Signal<T>) -> Signal<T>` 。

理解了 flatMap 和 map 的不同，实现起来也就很简单了：

```swift
func flatMap<T>(_ transform: @escaping (Value) -> Signal<T>) -> Signal<T> {
     let (sink, signal) = Signal<T>.empty()
     var _dispose: Disposable?
     let dispose = subscribe { (result) in
         switch result {
         case .success(let value):
             let new = transform(value)
             _dispose = new.subscribe({ _result in
                 sink(_result)
             })
         case .error(let error):
             sink(.error(error))
         }
    }
    if _dispose != nil {
        signal.objects.append(_dispose!)
    }
    signal.objects.append(dispose)
    return signal
}
```

现在我们可以模拟一个网络请求来测试 flatMap：

```swift
func users() -> Signal<[User]> {
     let (sink, signal) = Signal<[User]>.empty()
     DispatchQueue.main.asyncAfter(deadline: DispatchTime.now()+2) {
         let users = Array(1...10).map{ User(id: String(describing: $0)) }
         sink(.success(users))
     }
     return signal
 }
    
func userDetail(with id: String) -> Signal<User> {
    let (sink, signal) = Signal<User>.empty()
    DispatchQueue.main.asyncAfter(deadline: DispatchTime.now()+2) {
        sink(.success(User(id: id, name: "jewelz")))
    }
    return signal
}

let dispose = users()
    .flatMap { return self.userDetail(with: $0.first!.id) }
    .subscribe { result in
        print(result)
}
disposes.append(dispose)

// Print: success(ReactivePrograming.User(name: Optional("jewelz"), id: "1"))
```

通过使用 flatMap ，我们可以很简单的将一个 Signal 转换为另一个 Signal , 这在我们处理多个请求嵌套时就会很方便了。

## 写在最后

上面通过100 多行的代码就实现了一个简单的响应式编程库。不过对于一个库来说，以上的内容还远远不够。现在的 Signal 还不具有原子性，要作为一个实际可用的库，应该是线程安的。还有我们对 Disposable 的处理也不够优雅，可以模仿 RxSwift 中 DisposeBag 的做法。上面这些问题可以留给读者自己去思考了。（更多内容可以查看[我的主页](http://jewelz.me)）