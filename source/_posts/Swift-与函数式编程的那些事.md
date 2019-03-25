---
title: Swift与函数式编程的那些事
date: 2018-03-09 19:09:04
tags: Swift
categories: Swift
---

函数式编程所依赖的原理，在很多方面其实是早于编程本身出现的。因为函数式编程这种范式依赖于 Alonzo Church 在20世纪30年代发明的 [λ演算](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)。 λ 演算的一个核心思想是不可变性——某个符合所对应的值永远是不变的。所以从理论上来讲，函数式编程语言中应该是没有赋值语句的。

<!-- excerpt -->

本文是 Swift 系列文章中的第三篇，前两篇文章分别是：[Swift 与面向协议编程的那些事](http://jewelz.me/cjt0zq7ce0006620o0nhutb0j/)，[在 Swift 中使用值类型]()。按照计划，这篇文章主要介绍一下函数式编程思想在 Swift 中的应用。



函数式编程所依赖的原理，在很多方面其实是早于编程本身出现的。因为函数式编程这种范式依赖于 Alonzo Church 在20世纪30年代发明的 [λ演算](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)。 λ 演算的一个核心思想是不可变性——某个符合所对应的值永远是不变的。所以从理论上来讲，函数式编程语言中应该是没有赋值语句的。



函数式编程在维基百科中的定义是：**函数式编程**（functional programming）或称函数程序设计、泛函编程，是一种编程范式，它将计算机运算视为函数运算，并且避免使用程序状态以及易变对象。其中，[λ演算](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)为该语言最重要的基础。而且，λ演算的函数可以接受函数当作输入（引数）和输出（传出值）。

  

## 计算数组元素之和

我们最好还是用一个例子来解释什么是函数式编程。请看下面的这个例子：这段代码想要输出整型数组中所有元素的和。

```swift
func sum(of arr: [Int]) -> Int {
    var sum = 0
    for i in arr {
        sum += i
    }
    return sum
}

sum(of: Array(1...10)) // result: 55
```

下面我们改用 Swift 标准库中提供的函数来写这个程序，其代码如下：

```swift
Array(1...10).reduce(0, +) // result: 55
```

这里我们直接调用数组的 `reduce` 方法，该方法接受一个初始值和一个闭包（就是一个匿名函数），最终将结果返回。如果从来没有接触过函数式可能觉得这段代码看起来很奇怪。没关系，我们可以把这段代码改得完整一点，就像下面这样：

```swift
Array(1...10).reduce(0, { result, ele in
    return result + ele
})
```

`reduce` 方法第二个参数接受一个函数，为了简单起见，我们只管 Int 型数据，那么其形式可能是这样：`(Int, Int) -> Int` 。如果我们编写了一个像下面这样的函数：

```swift
func addTwoNums(x: Int, y: Int) -> Int {
    return x + y
}
```

然后，将它作为参数传给 `reduce` ，这样仍然能得到我们想要的结果，就像这样：

```swift
Array(1...10).reduce(0, addTowNums) // result: 55
```

因为Swift 标准库中 Int 实现了 `+` 运算符，它其实就是个函数。

```swift
public struct Int : FixedWidthInteger, SignedInteger {
	public static func + (lhs: Int, rhs: Int) -> Int
}
```

所以上面的代码中可以将 `+` 作为函数传给 `reduce`。如果你想实现数组元素相乘，那么你就可以把 `*` 作为函数传给 `reduce`。正是得益于 Swift 中函数式的特性，我们才能将 `+`, `-`, `*`, `/` 等普通操作符（在 Swift 中其实就是函数了）作为函数传给 `reduce`。



## 不可变性与软件架构

在上面的代码中，我们为什么说 `+` 具有函数式的特性？因为它符合函数式的一个核心思想：不可变性。对于 `+` 来说，它没有改变任何外部变量，而且不管你在什么地方，即使是并发环境下，只要传入的值一样，其结果永远都是一样的。如果一个函数，即使其返回结果永远不变，但是它改变了外部变量，它仍然不能说是函数式的。因为它仍然是可变的。



对于函数式编程，我们可以简单地归纳有以下特征：

* 只用 "表达式"，不用 "语句"

  "表达式"（expression）是一个单纯的运算过程，总是有返回值；"语句"（statement）是执行某种操作，没有返回值。函数式编程要求，只使用表达式，不使用语句。也就是说，每一步都是单纯的运算，而且都有返回值。

* 没有"副作用"

  所谓"副作用"（side effect），指的是函数内部与外部互动（最典型的情况，就是修改全局变量的值），产生运算以外的其他结果。

  函数式编程强调没有"副作用"，意味着函数要保持独立，所有功能就是返回一个新的值，没有其他行为，尤其是不得修改外部变量的值。

* 不修改状态

  上一点已经提到，函数式编程只是返回新的值，不修改系统变量。因此，不修改变量，也是它的一个重要特点。

  在其他类型的语言中，变量往往用来保存"状态"（state）。不修改变量，意味着状态不能保存在变量中。

* 引用透明

  函数程序通常还加强引用透明性，即如果提供同样的输入，那么函数总是返回同样的结果。就是说，表达式的值不依赖于可以改变值的全局状态。这使您可以从形式上推断程序行为，因为表达式的意义只取决于其子表达式而不是计算顺序或者其他表达式的副作用。这有助于验证正确性、简化算法，甚至有助于找出优化它的方法。

从以上函数式编程的特征来看，它们的共同作用最终导致一个结果：不可变性。

为什么不可变性是软件架构设计需要考虑的重点呢？为什么软件架构师要操心变量的可变性呢？答案显而易见：所有的竞争问题、死锁问题、并发问题都是由可变变量导致的。如果变量永远不会被更改，那么就不可能产生竞争或者并发问题。如果锁的状态是不可变的，那么永远就不会产生死锁问题。

在函数式编程中，由于数据全部都是不可变的，所以没有并发编程的问题，是多线程安全的。可以有效降低程序运行中所产生的副作用，对于快速迭代的项目来说，函数式编程可以实现函数与函数之间的热切换而不用担心数据的问题，因为它是以函数作为最小单位的，只要函数与函数之间的关系正确即可保证结果的正确性。



## map、flatMap 与函数式

map 是我们在使用数组是经常使用的方法，如果我们想将数组中的每个元素做个变换，就会使用到它。例如我们想将一个整形数组中的每个元素做平方操作，就可以这样：

```swift
let result = Array(1...10).map { $0 * $0 }
```

map 方法接受一个闭包作为参数，然后它会遍历整个数组，并对数组中的每个元素执行闭包中的操作，最后返回一个新数组，上面例子中将每个元素做平方，所以最后返的新数组就是：`[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]`

下面我们可以看一下 map 在 Array 中的定义：

```swift
func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
```

对 `rethrows` 关键字不是很了解的同学可以看看[这篇文章](http://jewelz.me/cjt0zq7cf0008620oknjdqsx7/)，这里我们可以忽略它，我们主要把关注点放到 `(Element) -> T` 这个闭包的定义上。我们可以看到，该闭包接受的参数类型跟我们数组中元素的类型是一致的，其返回类型跟我们最终想得到的数据中元素的类型是一样的。也就是说我们可以使用 map 方法将某个类型的数组转换成完全另一种类型的数组，例如下面这样：

```swift
let stringArr = Array(1...3).map { "No.\($0)" }  //["No.1", "No.2", "No.3"]

```

知道了 map 方法做的事情后，我们就很容易地实现我们的 map，代码可以像下面这样：

```swift
extension Array {
    func myMap<T>(_ transform: (Element) throws -> T) rethrows -> [T] {
        guard count > 0 else {
            return []
        }
        var result = [T]()
        for ele in self {
            result.append(try transform(ele))
        }
        return result
    }
}
```

这行代码 `result.append(try transform(ele))` 就是 map 方法的核心，这就是上面说的 map 对数组中的每个元素执行闭包中的操作。

如果你常用 Swift 的话， 还会发现除了数组定义了 map 方法， 同样 Optional 也存在这个方法。请看下面的代码：

```swift
var time: String? = "2018-01-01"

label.text = time.map{ "时间：\($0)"}  // "时间：2018-01-01"
```

上面的代码经常会出现在我的项目中。`time` 是定义在一个结构体或类中，它的值是由服务器返回的，所以对于它的值我们不能确定，所以一般我定义成可选型，最终我们要在界面上显示成 `时间：xxxx-xx-xx` 的样式，我们得在服务器返回的字符串前面加上 `时间：`，如果你直接 `"时间：\(time)"}` 肯定是不行的，因为 `time` 是可选型，在使用时你得先解包，这样我们就得写一串处理 `time` 的代码（当然这里处理代码也很短，其它情况可能就比较长了），这样看起来非常繁琐。使用 `map` 方法就使得我们的代码干净简洁了很多。

我们可以看一下 Optional 中 map 的定义：

```swift
func map<U>(_ transform: (Wrapped) throws -> U) rethrows -> U?
```

你再回去看一下 Array 中 map 的定义，你会发现二者几乎没有区别。Array 中 返回的是一个数组 ，这里返回的是 可选型的值，这里看似没有任何联系，如果你把数组和可选型当成一种包装类型，你会发现它们是一样的，所以才有了 `map` 这个相同的行为。

那么什么是包装类型的值呢，你可以简单地理解为包含了多个值的一种值，例如数组，你可以通过一个数组变量访问到数组中的任何一个值，而对于可选型，也是一样的，你可以访问到一个 nil 值，或者一个解包后的值。像标准库中的 `String`, `Dictionary` 等都可以看做包装值，而且它们都有实现 `map`。



flatMap 在使用上和 map 非常相似，如果你不仔细观察的话，你甚至都发现不了它们之间的区别。我们先看一下 flatMap 在 Optional 中的定义：

```swift
func flatMap<U>(_ transform: (Wrapped) throws -> U?) rethrows -> U?
```

你仔细看的话，你会发现它与 map 的区别，flatMap 接受的闭包的返回类型是 `U?`，而 map 中的是 `U`，这就是它们在方法签名上唯一的区别。也就是说，map 中闭包返回值不能是可选型，而 flatMap 可以。如果你把它替换成上面包装值的概念，那就是 flatMap 中的闭包的返回值也是个包装值。

我们可以看一下数组中 map 和 flatMap 的区别是怎样的：

```swift
func map<T>(_ transform: (Element) throws -> T) rethrows -> [T]
func flatMap<SegmentOfResult>(_ transform: (Element) throws -> SegmentOfResult) rethrows -> [SegmentOfResult.Element] where SegmentOfResult : Sequence
```

虽然 flatMap 的方法签名比 map 复杂了很多，但是主要区别也是体现在闭包返回值类型上，map 中返回值是 T 的单一类型，而 flatMap 返回的是一个 Sequence，这里你可以简单理解为数组。对比上面 Optional 中 map 和 flatMap 的区别，你会发现它们都区别是一致的。

运行下面的代码，你就能更清楚的看到它们都区别了：

```swift
let array = [[1, 2], [3, 4]]
let mapedArr = array.map { $0.map{ $0 * $0} }          // [[1, 4], [9, 16]]
let flatMapedArr = array.flatMap { $0.map{ $0 * $0} }  // [1, 4, 9, 16]
```



### 为什么这样设计

看到这里， 大家可能会产生疑问了。 为什么会多出个 flatMap 函数？这其实涉及另外一个维度的概念， Functors 和 Monads。 明白这个概念之后，你就会发现这其中的关联，以及为什么会有 map 和 flatMap 这两个函数存在了。

#### Functors

在讲 functors和 monads 时，我们需要用到上面讲 Array 和 Optional 联系时用到的包装值的概念。

其实 简单来说，Functors 就是将包装值直接传递给函数的一种行为。 对应到我们的代码上，就是 map 函数了, 看下面这个例子：

```swift
func square(_ val: Int) -> Int {
    return val * val
}

square(10) // 100
var optionalVal: Int? = 10
square(optionalVal) // Value of optional type 'Int?' not unwrapped; did you mean to use '!' or '?'?
```

上面的代码中，我们想求一个整数的平方，对于一个可选类型的值来说，我们必须将它解包后，才能传入 `square` 函数中。我们虽然不能直接把 optionalVal 直接传递给 `square` 函数，但是我们可以使用 map 将 `square` 最为参数传递进来，这就相当于间接地将 optionalVal 传递给 `square` 函数。就像下面代码中那样：

```swift
optionalVal.map(square)
```

同样地，我们把数组也看做一个包装值，虽然不能直接将数组传递给 `square` 函数，但是我们使用 map 仍然能将数组中的值传递给 `square` 函数。

```swift
Array(1...10).map(square)
```

总的来说 Functors 就是将一个**包装值**直接传递给函数，并且返回的结果依然是包装值的一种行为。 我们调用 Optional 中的 map 函数， 会用闭包将 Optional 中的值进行操作，然后返回值还是一个 Optional。

同样，我们对数组调用 map 函数， 会用闭包将数组中的值进行一些操作， 然后返回值还是一个数组。

从这个维度来思考，就能理解为什么 Optional 和数组，这两个看似没有任何关联的类型，为什么都有 map 和 flatMap 方法了。

#### Monads

如果说 Functors 对应的是 map 函数， 那么 Monads 对应的就是 flatMap 函数啦。

Monads 用一句话来说就是， 它将一个**包装值**传递给一个返回值类型是**包装值**的函数。注意 monads 强调的是**返回值类型是包装值的函数** 。

Optional 的 flatMap 函数接受的闭包是 (Wrapped) -> U?， 它返回的还是 Optional 类型。 数组的 flatMap 接受的闭包是 (Element) -> SegmentOfResult，这里 SegmentOfResult 必须是 Sequence， 返回的依然还是数组。

map 和 flatMap 的主要区别就是他们所接受闭包的返回类型， map 的闭包返回的是一个普通值， flatMap 的闭包返回的是一个包装值。

下面给出一个 monads 在实际中运用的例子：在下面代码中，在第一个请求返回后，我们拿到结果并发起第二个请求。

```swift
Provider<UserApi>(.users)
    .flatMap { response -> Provider<UserApi> in
     	let res = response.array.first as! [String: Any]
        let user = User(res)
        return Provider(.detail(user.name))
    }
    .request { response in
    	print(response)
    }.addToCancelBag()
```

这里的 Provider 就是一个包装值，它的 flatMap 接受的闭包的返回类型就是另一个包装值。这完全符合 monads 的定义。



Functors 和 Monads 并不是 Swift 中独有的，它们是一种数学概念。而在函数式编程中，你经常会看到它们都身影。只有是**把包装值传递给函数，并且返回的结果依然是包装值**的行为就是 functors，**把包装值传递给一个返回值类型是包装值的函数**的行为就是 Monads。



## ?? 与函数式

如果你对 Swift 中的可选类型 (Optional) 用的比较多的话，那么你可能会经常用到 `??` 这个操作符，就像下面这样：

```swift
var a: Int?
print("a =", a ?? 100)
```

`??` 操作符左边是一个 Optional值，右边是一个普通值，它的作用就是，如果左边的 Optional 值为 nil， 那么就使用右边的值作为结果，如果左边的 Optional 不为 nil，则返回左边的 Optional 解包后的值，就像下面代码展示的那样：

```swift
var a: Int?
print("a =", a == nil ? 100 : a!)
```

事情真的是这么简单吗？在回答这个问题前，我们可以先自己实现一个 `??`，为了跟系统的进行区分，这里我们把新函数定义为 `???`，为了实现我们的 `???` 函数，就必须使用自定义操作符。最终的代码就像这样：

```swift
infix operator ???: AdditionPrecedence

func ??? <T> (optional: T?, defaultValue: T) -> T {
    guard let value = optional else { return defaultValue }
    return value
}

print("a =", a ??? 100) // 100
```

目前看起来好像跟标准库中 `??` 的结果是一模一样的，不过先不要着急，我们可以把 `??` 和我们的 `???` 右边替换成一个函数，代码是这样的：

```swift
func doSomething() -> Int {
    var sum = 0;
    for i in 0...10000 {
        sum += i
    }
    return sum
}

var a: Int? = 1
var b1 = clock()
let result = a ?? doSomething()
print("time1: ", clock() - b1)

var b2 = clock()
let result2 = a ??? doSomething()
print("time2: ", clock() - b2)
```

为了测试  `??` 和我们的 `???`的性能，我简单地记录了这两个调用的时间，下面是其中一次的结果：

```
time1:  206
time2:  254042
```

不过你执行多少次，你都能得出一个结果：`??` 调用比`???` 调用快了很多。为什么会出现这种情况，我们的 `???` 到底有什么问题呢？如果你在 `doSomething` 函数中打个断点，你会发现，它只在 `???` 调用时才会执行。也就是说 `??` 函数在判断了第一个值不为 nil 时，不会调用第二个表达式。而我们在调用 `???` 之前，就必须先把右边的数据准备好，这样不管左边的值是不是 nil，右边的表达式都会调用。所以才有了上面的区别。那么系统是怎么做到这样的呢？我们很容易就想到，把右边的值替换成一个函数，就可以办到。代码就像这样：

```swift
func ??? <T> (optional: T?, defaultValue: () -> T) -> T {
    guard let value = optional else { return defaultValue() }
    return value
}

let result2 = a ??? doSomething
```

再次运行，你会发现 `doSomething` 函数没有再执行了。细心的同学可能会发现，这里调用 `???` 跟调用 `??` 有点区别，在 `??` 的调用中，右边是 `doSomethin()`，而我们的 `???` 右边是 `doSomething`。这里其实使用了一个叫做自动闭包的东西，我们在闭包前面加上 `@autoclosure` 关键字，就能让我们的普通闭包变成自动闭包。使用自动闭包的好处是编译器会自动把一个值转成闭包。当然，自动闭包是有一定限制的，只能作用在没有参数的闭包上。我们把代码修改一下就可以跟调用 `??` 一样，使用我们的 `???` 了。

```swift
func ??? <T> (optional: T?, defaultValue: @autoclosure () -> T) -> T {}
```



这里花这么多时间讲 `??` 操作符，其实是为了体现函数式的另一个特征，那就是惰性计算。从上面的例子中，可以看出，有与没有惰性计算，在程序性能上还是有很大差别的。上面只是简单的做了 10000 次循环，运行时间就差了百倍。如果这是个耗时操作，那么这种优化就可以大大地提高程序性能。

在惰性计算中，表达式不是在绑定到变量时立即计算，而是在求值程序需要产生表达式的值时进行计算。延迟的计算使您可以编写更高性能和避免可能潜在地生成无穷输出的函数。因为它不会计算多于程序的其余部分所需要的值，所以不需要担心由于表达式执行耗时操作所带来的性能问题或由无穷计算所导致的 out-of-memory 错误。



上面就是我对函数式编程的一些思考，因为篇幅有限，很多东西不能展开来讲。如果同学们能从我这篇文章中得到一点启发，那么这篇文章也就能提现它的价值了。