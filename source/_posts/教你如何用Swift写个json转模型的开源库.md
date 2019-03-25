---
title: 教你如何用Swift写个json转模型的开源库
date: 2017-03-26 17:32:56
tags: Swift
category: Swift
thumbnailImage: thumbnail.jpg
---
在iOS项目开发过程中，我们经常会用到将从服务器获取的 json 转 model 的操作，我们可以使用 Swift 提供的`setValuesForKeys` 或者 Objective-C 提供的`setValuesForKeysWithDictionary` 方法来完成这一操作。

<!--more-->

使用上面两个方法只能将字典转换成 model , 如果 json 最外层是个数组，那么我们就必须在循环中使用这个方法，这非常不方便， 而且还有个条件，就是 model 中的所有属性名必须跟字典中的 key 完全对应，这样就会遇到另外一个问题，如果我们字典中的一个 key 与系统关键字重名，那我们在 model 就不能使用这个 key 作为属性名了。



为了解决上面的问题，我们会使用一些第三方库去完成字典转模型的操作，例如 MJExtension 。由于它是一个 OC 的库，所以在 Swift 项目中需要引入桥接文件。在 Swift 中使用其 API 时其实是很不 swift 的。所以现在我们就用 Swift 3.0 来写一个 swift style 的 json 转模型的库吧。

例如我们有这样的两个 model:
```
class User: NSObject {
    var name: String?
    var age = 0
    var desc: String?
}
class Repos: NSObject {
    var title: String?
    var owner: User?
    var viewers: [User]?
}
```
最终我们想实现这样的调用: 
```
let repos = json ~> Repos.self    // 将一个字典转成一个Repos的实例
 
let viewers  = viewers => User.self  //将一个数组转换成User的数组
```
`~>` 和 `=>` 是自定义的运算符，主要是为了调用方便。它们的定义是这样的：
```
public func ~><T: NSObject>(lhs: Any, rhs: T.Type) -> T?
public func =><T: NSObject>(lhs: Any, rhs: T.Type) -> [T]?
```
这里给出我的实现 [ModelSwift](https://github.com/hujewelz/modelSwift)。大家可以先看看我的实现然后试着写出自己的实现。好了，现在就让我们开始吧。

## 要解决的问题
由于将数组转成模型数组，其实要做的工作跟将字典转模型是一样的，只是做了个循环而已。所以我们首先要解决的问题是：如何在 Swift 将字典转成模型。这里我们是使用 KVC就可以了。我们使用 NSObject 的  `setValue(_ value: Any?, forKey key: String)` 方法来给对象设置值。

从上面要实现的效果来看，我们在使用前并不用先实例化一个对象。所以我们要解决的第二个问题是：如何通过类型来实例化一个对象。 

另一个要解决的问题是字典中的 key 与关键字重名，或者我们想使用自己的名字。所以我们要实现自己的映射的策略。

还有一个问题是，如果我们服务器返回的字典数据中包含另外一个字典数组，对应我们的 model 中就是一个对象包含另外一个对象的数组。那么我们怎样才能知道这个数组中对象的类型呢？

## 实现思路
对于上面提到的第一问题我在上面已经给出了解决方案，就是让我们的 model 继承 NSObject, 然后使用   `setValue(_ value: Any?, forKey key: String)` 方法来给对象设置值。这里的 `value` 其实是要根据 model 中的属性名去字典中获取的。如果我们能拿到 model 所有的属性名，那么给 model 设置值的操作就完成了。那么如何获取到 model 的属性名呢？这就必须得使用到 Swift 中的反射机制了。

### Mirror
Swift 的反射机制是基于一个叫 Mirror 的 `struct` 来实现的。对于 Mirror 的详细结构大家可以按住 `cmd` 点进去查看。这里我们主要关注的是 `public typealias Child = (label: String?, value: Any)` 这个 typealias，它其实是一个元祖，`label` 就表示我们的属性名，是 Optional 的。`value` 表示的是属性的值。这里 ` label` 为什么是 Optional 的？如果你仔细考虑下，其实这是非常有意义的，并不是所有支持反射的数据结构都包含有名字的子节点。 Mirror 会以属性的名字做为 `label`，但是 Collection 只有下标，没有名字。Tuple 同样也可能没有给它们的条目指定名字。

Mirror 有个 `children` 的存储属性，它的定义是这样的: 
```
 public let children: Mirror.Children
```
这里的 `Mirror.Children` 也是一个 typealias，它是这样定义的：
```
public typealias Children = AnyCollection<Mirror.Child>
```
可以看到它是 Child 的集合。所以我们可以通过 Mirror 的 `children` 属性来获得 model 的所有属性名。

我们写个类来测试一下：
```
class Person: NSObject {
    var name = ""
    var age = 0
    var friends: [Person]?
}

let mirror = Mirror(reflecting: Person())
for case let (label?, value) in mirror.children {
    print ("\(label) = \(value)")
}
```
运行结果是如下：
```
name = 
age = 0
friends = nil
```
Mirror 还有一个类型为 `Any.Type` 的 `subjectType` 存储属性，表示该映射对象的类型，例如上面的 `mirror.subjectType` 就是 `User`。使用 `subjectType` 就可以获得对象的类型以及其所有属性的类型。为了实现这个效果，我们可以写出下面的代码:
```
func subjectType(of subject: Any) -> Any.Type {
    let mirror = Mirror(reflecting: subject)
    return mirror.subjectType
}

func children(of subject: Any) {
    let mirror = Mirror(reflecting: subject)
    for case let(label?, value) in mirror.children {
        print ("\(label) = \(subjectType(of: value))")
    }
}

children(of: Person())
```
打印结果是这样的：
```
name = String
age = Int
friends = Optional<Array<Person>>
```
我原本想使用这个方法来得到 model 中包含的另外对象的类型和数组中对象的类型，例如 Person 中有 `father` 和 `friends` 属性:
```
class Person: NSObject {
    var name = ""
    var age = 100
    var father: Person?
    var friends: [Person]?
}
```
但是发现结果是 `Optional<Person>` 和 `Optional<Array<Person>>`。所以我们还是得显示地指出一个 model 中包含的其他对象的类型，以及数组中对象的类型。在后面我会给出自己的实现。大家可以给出自己的实现。

### 通过类型来实例化一个对象
要使用 Mirror 来获得反射对象的所有属性名，就必须先使用 `init(reflecting subject: Any)` 来创建一个 Mirror。而创建 Mirror 就必须传入一个 subject（在这里我们主要传入一个NSObject类型的对象）。所以我们的首要任务就是通过类型来实例化一个对象。
> 有些同学可能有疑问了：我要转换成 Person 的对象，我直接传入一个
 Person 的实例就行了啊。如果你看看我们 josn 转模型的方法定义就能明白了。 ` func ~><T: NSObject>(lhs: Any, rhs: T.Type) -> T?`

还是以上面的 Person 为例，我们看看这样的调用:
```
Person.self().age
// 结果是：100
```
所以我们通过一个类的 `self()`方法可以得到一个类的实例。其实我们还可以通过 AnyClass 来实例化对象。AnyClass 是类的类型，其定义是这样的：
```
public typealias AnyClass = AnyObject.Type
```
我们通过类的`self`属性可以得到类的类型：
```
Person.self     
//结果是：Person.Type
```
得到类的类型后，通过调用其 `init()`方法就可以创建一个实例了：
```
Person.self.init().age
// 结果是：100
```
> 使用类型创建对象的类中的init方法前面必须是 required 的，因为这么创建方式是使用meta type来创建的。由于我们 json 转模型的 model 是继承自 NSObject 的，所以不用在每个类中显示地实现。

## 写个简单的 josn 转模型
有了上面的基础就可以来实现我们的 josn 转模型了。首先我们来写出 `~>` 的定义，并通过类来创建一个对象
```
infix operator ~>

func ~><T: NSObject>(lhs: Any, rhs: T.Type) -> T? {
    guard let json = lhs as? [String: Any], !json.isEmpty else {
        return nil
    }
    
    let obj = T.self()
    let mirror = Mirror(reflecting: obj)
    
    for case let(label?, value) in mirror.children {
        print ("\(label) = \(value)")
    }
    
    return obj
}

class Person: NSObject {
    var name = ""
    var age = 0

    override var description: String {
        return "name = \(name), age = \(age)"
    }
}
let json: [String: Any] = ["name": "jewelz", "age": 100]
let p = json ~> Person.self
// 打印结果：
// name = 
// age = 0
```
通过上面的几行代码我们确实成功的创建了一个 Person 的实例了。下一步就是给实例设置值了。我们在上面的 `for` 循环中添加如下代码：
```
// 从字典中获取值
if let value = json[label] {
     obj.setValue(value, forKey: label)
}
```
整个代码就是这样的：
```
infix operator ~>

func ~><T: NSObject>(lhs: Any, rhs: T.Type) -> T? {
    guard let json = lhs as? [String: Any], !json.isEmpty else {
        return nil
    }
    
    let obj = T.self()
    let mirror = Mirror(reflecting: obj)
    
    for case let(label?, _) in mirror.children {
        // 从字典中获取值
        if let value = json[label] {
            obj.setValue(value, forKey: label)
        }
    }
    return obj
}

let p = json ~> Person.self
print(p!)
//结果：name = jewelz, age = 100
```
有了上面 `~>` 的实现，`=>` 的实现就很简单了：
```
infix operator =>
func =><T: NSObject>(lhs: Any, rhs: T.Type) -> [T]? {
    guard let array = lhs as? [Any], !array.isEmpty else {
        return nil
    }
    
    return array.flatMap{ $0 ~> rhs }
}
```
上面只是实现了一个简单 josn 转模型，其实在实际项目中要解决的问题还有很多。现在再来看看我在文章开头给出的 User 类和 Respo 类:
```
class User: NSObject {
    var name: String?
    var age = 0
    var desc: String?
}
class Repos: NSObject {
    var title: String?
    var owner: User?
    var viewers: [User]?
}
```
只简单的用上面的实现是无法得到想要的结果的。对于 User 类来说，`desc` 属性对应 json 的 `description` key，所以我们还要进行 model 的属性与 json 的键的映射。这里的思路就是将 model 的属性名作为 key，以要替换的 json 的键作为 value 存入字典中。我们可以拓展 NSObject ，添加一个计算属性并提供一个空实现。不过这样的倾入性太大，毕竟不是所有的类都需要做这个映射。所以最后的方式是 POP。比如我们可以制定这样一个协议：
```
public protocol Reflectable: class {
    var reflectedObject: [String: Any.Type] { get }
}
```
在需要做映射的类中去实现该协议。

对于更复杂的 Repos 类来说，要做的事情更多。比如  `owner`的类型怎么知道？`owner` 这个对象怎么完成赋值？`viewers` 数组中的类型是什么，怎样才能完成赋值？ 虽然通过上面提到的 Mirro 可以得到所有的类型，但得到的是 `Optional<User>`以及 `Optional<Array<User>>`。我的解决的办法就跟上面做属性名替换是一样的。这里就不详细地说明了，大家可以各显神通。写出自己的实现。

## 写在最后
通过上面的几个步骤，我们就能很快的实现一个简单的 json 转模型的需求了。总结起来就是以下几点：

* 所有要转换的 model 继承 NSObject 
* 使用类的类型来实例化对象
* 通过反射获得对象的所有属性名
* 通过  `setValue(_ value: Any?, forKey key: String)` 方法来给属性设置值


对于在最后提出的几个问题，我这里就不一一详细地说明了。大家可以[点这里](https://github.com/hujewelz/modelSwift)看看我的实现。大家可以使用 CocoaPods 或者 Carthage 将 [ModelSwift](https://github.com/hujewelz/modelSwift) 集成到项目中。如果在使用中有什么问题可以 issue 我，也可以给个 star 持续关注。

