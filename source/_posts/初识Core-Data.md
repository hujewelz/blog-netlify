---
title: 初识Core Data
date: 2015-09-27 14:40:29
tags: 
  - iOS 
  - Core Data
category: iOS
---
Core Data 是一个强大的对象图形化管理和对象持久化的框架，这一框架在 iOS 和 OS X 系统中已经存在很多年了。<!-- more --> 2005年的四月份，Apple 发布了 OS X 10.4，正是在这个版本中 Core Data 框架发布了。Core Data 可以很方便地将 `OC` 对象存储到数据库中，也可以将数据库中的数据转化为 `OC` 对象，在这个过程中不需要我们手动编写任何 `SQL` 语句，Core Data 会帮我们完成。对于不喜欢 `SQL` 语句的同学来说，使用 Core Data 倒是个不错的选择。即使你不愿使用 `Core Data` ，作为一个合格的 iOS 开发者， 你也应该熟悉 Core Data。



## 组成部分
在 Core Data 中有几个比较重要的类：

* **NSManagedObjectContext**
    托管对象上下文，我们进行数据操作时，大多都是和它打交道。我们每一个托管对象都存在于一个 context 内。Core Data 支持多个 contexts，不过对于更高级的使用情况才用。
* **NSManagedObjectModel**
    托管对象模型，我们一般通过 `.xcdatamodeid`文件来加载一个托管对象模型，也可以通过代码创建。你可以把它理解为一个数据库。
* **NSPersistentStoreCoordinator**
    持久化存储协调器（persistent store coordinator），它将对象图管理部分和持久化部分捆绑在一起，当它们两者中的任何一部分需要和另一部分交流时，这便需要持久化存储协调器来调节了。
* **NSPersistentStore**
    持久化存储（persistent store），每个持久化存储协调器都有一个属于自己的持久化存储。它在文件系统中与 SQLite 数据库交互。为了支持更高级的设置，Core Data 可以将多个 stores 附属于同一个持久化存储协调器，并且除了存储 SQL 格式外，还有很多存储类型可供选择。
* **NSManagedObject**
    托管对象类（Entity），所有 Core Data 中的托管对象都必须继承该类，根据实体创建托管对象类文件。

下图很清晰地表现了它们之间的关系：

![](https://objccn.io/images/issues/issue-4/stack-complex.png)

当所有的组件都捆绑到一起的时候，我们把它称作 Core Data 堆栈，这个堆栈有两个主要部分。一部分是关于对象图管理，这正是我们需要很好掌握的那一部分，并且知道怎么使用。第二部分是关于持久化，也就是数据如何存储到磁盘中。持久化存储协调器（persistent store coordinator）刚好位于堆栈中间。这样很好的将两部分实现了分离，我们就不用关心存储的实现细节。

## 创建 Core Data 堆栈
创建一个 Core Data 堆栈最方便快捷的方式是在我们创建项目时选择 `Use Core Data` 复选框，这样创建出来的工程系统会默认生成一些CoreData的代码以及一个.xcdatamodeld后缀的模型文件。

![](http://image18-c.poco.cn/mypoco/myphoto/20170227/15/18436043320170227154154030.png?687x133_130)

不过我建议不要这么做，因为 Xcode 会把自动生成的部分代码放在AppDelegate中，我们应该把这部分代码单独抽离出去，放在专门的类或模块来管理 Core Data 相关的逻辑。

### 构建模型文件
使用 Core Data 的第一步是创建后缀为 `.xcdatamodeld`的模型文件，使用快捷键 Command + N，选择 Core Data -> Data Model -> Next，完成模型文件的创建。创建完成后，点击底部 `Add Entity` 按钮，来添加一个实体，命名为 `User`。

添加 `User` 实体后，会发现一个实体对应着三部分内容：Attributes、Relationships、Fetched Properties，分别对应着属性、关联关系、获取操作。

![](http://image18-c.poco.cn/mypoco/myphoto/20170227/15/18436043320170227155859067.png?1138x620_130)

点击 Attributes 下面的加号按钮可以给实体添加属性。
![](http://image18-c.poco.cn/mypoco/myphoto/20170227/16/18436043320170227160223054.png?921x132_130)

### 设置堆栈
我们使用 `initWithConcurrencyType:` 为主队列创建一个 managed object context，在有些代码中，你可能见到 `[[NSManagedObjectContext alloc] init]`。不过最好使用 `initWithConcurrencyType: `初始化，以明确你是使用基于队列的并发模型。

```objectivc-c
- (NSManagedObjectContext *)managedObjectContext {
  if (_managedObjectContext == nil) {
    _managedObjectContext = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
    _managedObjectContext.persistentStoreCoordinator = self.persistentStoreCoordinator;
  }
  return _managedObjectContext;
}
```
在创建 managed object context 时我们给它设置了一个持久化存储协调器
```
- (NSPersistentStoreCoordinator *)persistentStoreCoordinator {
  if (_persistentStoreCoordinator == nil) {
    NSURL *path = [[self applicationDocumentsDirectory] URLByAppendingPathComponent:@"test.sqlite"];
    _persistentStoreCoordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:self.objectModel];
    NSError *error = nil;
    if (![_persistentStoreCoordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:path options:nil error:&error]) {
      NSLog(@"error: %@", error.localizedDescription);
    }
  }
  return _persistentStoreCoordinator;
}

- (NSManagedObjectModel *)objectModel {
  if (_objectModel == nil) {
    NSURL *momdURL = [[NSBundle mainBundle] URLForResource:@"coredata" withExtension:@"momd"];
    _objectModel = [[NSManagedObjectModel alloc] initWithContentsOfURL:momdURL];
  }
  return _objectModel;
}
```
我们通过一个托管对象模型创建了一个持久化存储协调器，并使用 `addPersistentStoreWithType: configuration: URL: options: error:` 关联了数据库的部分，关联本地数据库后会返回一个 `NSPersistentStore` 对象，这个对象负责具体持久化存储的实现。可以多次调用该方法添加多个持久化存储对象。

一个持久化存储协调器有四种可选的持久化存储方案，用得最多的是 SQLite 的方式。其中Binary和XML这两种方式，在进行数据操作时，需要将整个文件加载到内存中，这样对内存的消耗是很大的。

* **NSSQLiteStoreType：**SQLite数据库
* **NSXMLStoreType：**XML文件
* **NSBinaryStoreType：**二进制文件
* **NSInMemoryStoreType：**直接存储在内存中

我在一个单独的类中，完成了以上所有操作，并添加了一个 `saveContext` 方法，方便在其他类中调用。
```objectivc-c
@interface DBHelper : NSObject

+ (instancetype)sharedInstance;

@property (nonatomic, strong, readonly) NSManagedObjectContext *managedObjectContext;
@property (nonatomic, strong, readonly) NSManagedObjectModel *objectModel;
@property (nonatomic, strong, readonly) NSPersistentStoreCoordinator *persistentStoreCoordinator;

- (void)saveContext;

@end
```
### 创建托管对象
创建实体后，就可以根据对应的实体，生成开发中使用的基于 NSManagedObject 类的托管对象类文件。使用快捷键 Command + N，选择 Core Data -> NSManagerObject subclass -> Next，选择模型文件 -> 选择实体，完成模型文件的创建。
![](http://image18-c.poco.cn/mypoco/myphoto/20170227/16/18436043320170227162725013.png?714x506_130)

## 操作数据
### 插入数据
在模型类中可以加入一个类方法来将新的对象插入到 managed object 上下文中，并使用 `saveContext` 将数据保存到SQLite数据库中：
```objectivc-c
User *user = [NSEntityDescription insertNewObjectForEntityForName:@"User" inManagedObjectContext:[DBHelper sharedInstance].managedObjectContext];
user.name = @"张三";
user.age = @22;
[[DBHelper sharedInstance] saveContext];
```
Core Data 创建新对象的 API 并不是非常的直观，我们可以以一种更加优雅的方式来实现同样的功能：
```objectivc-c
+ (NSString *)entityName{
  return NSStringFromClass(self);
}

+ (instancetype)insertNewObjectForEntity {
  return [NSEntityDescription insertNewObjectForEntityForName:[self entityName]
                                       inManagedObjectContext:[DBHelper sharedInstance].managedObjectContext];
}

- (void)save {
  [[DBHelper sharedInstance] saveContext];
}
```
现在保存一个对象就简单多了：
```objectivc-c
User *user = [User insertNewObjectForEntity];
user.name = @"李四";
user.age = @22;
[user save];
```
### 查询数据
我们只需要创建一个 `NSFetchRequest`对象，然后调用 managed object context 的 `executeFetchRequest:` 方法返回查询结果集合。

```objectivc-c
NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
  
NSError *error = nil;
NSArray *result = [[DBHelper sharedInstance].managedObjectContext executeFetchRequest:request error:&error];
if (error) {
    NSLog(@"error: %@", error.localizedDescription);
    return;
}
[result enumerateObjectsUsingBlock:^(User   * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    NSLog(@"name: %@, age: %zd", obj.name, obj.age.integerValue);
}];
```
我们通过给 `request` 设置一些条件，查询我们想要的数据：
```
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"name = %@", @"张三"];
request.predicate = predicate;
```
### 修改数据
修改数据很简单，我们只要根据条件查询出数据后，修改对象的属性后，保存数据即可：
```objectivc-c
NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"name=%@", @"张三"];
request.predicate = predicate;
  
NSError *error = nil;
NSArray *result = [[DBHelper sharedInstance].managedObjectContext executeFetchRequest:request error:&error];
if (error) {
    NSLog(@"error: %@", error.localizedDescription);
    return;
}
[result enumerateObjectsUsingBlock:^(User   * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    obj.age = @33;
}];
  
[[DBHelper sharedInstance] saveContext];
```

### 删除数据
删除数据跟修改数据几乎一模一样，唯一的区别就是查询出数据后，调用 managed object context 的 `deleteObject:` 方法来删除数据：
```objectivc-c
NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"User"];
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"name=%@", @"张三"];
request.predicate = predicate;
  
NSError *error = nil;
NSArray *result = [[DBHelper sharedInstance].managedObjectContext executeFetchRequest:request error:&error];
 if (error) {
    NSLog(@"error: %@", error.localizedDescription);
    return;
}


[result enumerateObjectsUsingBlock:^(User   * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    [[DBHelper sharedInstance].managedObjectContext deleteObject:obj];
 }];
  
[[DBHelper sharedInstance] saveContext];
```
## 总结
很多人，特别是初学者都认为 Core Data 很难，所以尽量去避免在项目中使用它。其实去了解后发现其实并不是很复杂。像上面的增删改查操作，看上去大体流程都差不多，都是一些最基础的简单操作。要想更深入地了解 Core Data 可以去网上看高级教程。


