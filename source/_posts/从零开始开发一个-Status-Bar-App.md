---
title: 开发你的第一个 Mac 应用
date: 2018-05-28 18:56:36
tags: 
  - Cocoa
  - Swift 
categories: Cocoa Programing
thumbnailImage: thumbnail.png
coverImage: cover.png
coverMeta: out
---



以前我们都是开发 iOS 应用，今天让我们来做点不一样的，来开始开发 Mac OS  应用。

<!--excerpt-->

> 由于云存储过期了，导致图片都无法显示了。

以前我们都是开发 iOS 应用，今天让我们来做点不一样的，来开始开发 Mac OS  应用。很多同学可能会觉得开发 Mac 应用是不是很挺难，不管你以前有没有 Mac 应用的开发经验，只有你按照下面的步骤来，你很快就能开发一款 Mac 应用了。



今天要做的不是普通的窗口应用，而是 Status Bar 应用。什么是 Status Bar 应用？就像下图中的就是 Status Bar 应用。这是不是比做一个窗口应用有趣多了。现在就开始吧。 



![](http://othizsxsl.bkt.clouddn.com/macapp00.png)



# Let`s beging



首先打开你的 xCode 创建一个新工程，记住要选择 macOS，创建一个 Cocoa App。



![](http://othizsxsl.bkt.clouddn.com/macapp01.png)

直接点击下一步。给你的工程取一个牛B哄哄的名字。有一点需要注意的是，Use Storyboards 和 Create Document-Based Application 单选框都不要选。

![](http://othizsxsl.bkt.clouddn.com/macapp02.png)

语言就选择你最熟悉的语言就可以了，我这里选的是 Swift。如果你还没开始使用 Swift，那么我强烈建议你最好开始使用 Swift 来开发你的新应用。

选好你要将工程存放的位置，点击 Create 就完成了工程的创建了。

工程创建完后，你会发现有一个 MainMenu.xib 文件。打开它，这就是你的应用默认的样子了。

![](http://othizsxsl.bkt.clouddn.com/macapp03.png)

你可以把 Objects 下面的 Main Menu 删掉，因为我们这个 App 中根本就用不到它，后面我们会添加自己的 Menu。

先暂时放下这个 xib 文件，让我们开始实现 Status Bar App吧。



# Status Item

打开 AppleDelegate.swift 文件，让我们开始编写 Mac App 的第一行代码吧。在你的 AppleDelegate 类中创建一个 NSStatusItem：

```swift
lazy var statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)
```

 然后在 `applicationDidFinishLaunching  `  添加以下代码：

```swift
func applicationDidFinishLaunching(_ aNotification: Notification) {
    let image = NSImage(named: NSImage.Name("Icon_32x32"))
    statusItem.highlightMode = true
    statusItem.image = image
}
```

你需要把 `NSImage(named: NSImage.Name("Icon_32x32"))` 中的图片名换成你自己的图片，并且别忘了把图片导入到项目中。现在你可以运行你的项目了。你会发现系统菜单栏中确实出现了刚刚你添加的 StatusItem，不过同时也出现了一个窗口，并且 Dock 栏中也显示了应用图标。这在我们的 Status Bar 应用中是不应该出现的。现在就先解决这两个问题。

要让我们的应用在运行时不要出现窗口很简单，你只需要打开 MainMenu.xib 文件，然后选中 Window，在右边的辅助编辑器中将 Visible At Launch 的单选框的勾选去掉就可以了。

![](http://othizsxsl.bkt.clouddn.com/macapp04.png)



要让我们的应用图标从 Dock 栏中去掉，你只需要在 Info.plist 文件中添加一行 `Application is agent (UIElement)`，并将其设置为 YES 即可。再次运行你会发现不会再出现窗口了，应用图标也从Dock 栏中去掉了。是不是很赞呢。

现在点击菜单栏中的图标，没有任何功能，现在是时候给我们的应用添加一些功能了，就像上面图片中那个样子。



# 添加 Menu



再次打开 MainMenu.xib 文件，在右侧辅助编辑器的下面搜索 menu，然后将一个 Menu 拖动到 xib 中，它会自动出现在 Objects 下。

![](http://othizsxsl.bkt.clouddn.com/macapp05.png)

默认情况下，Menu 中有三个 Menu Item，你可以将一个 Menu Item 拖到 Menu 中来添加更多的 Menu Item，当然，你也可以选中某一个，然后点击 `Delete` 键将其删掉。

这是我的菜单最终的样子:

![](http://othizsxsl.bkt.clouddn.com/macapp06.png)

鼠标双击一个菜单栏左侧位置，你就可以给菜单栏添加标题了，双击一个菜单栏右侧位置，然后在键盘中键入你想输入的键，作为菜单栏功能的快捷键。

为了让菜单更好看，我在第二个菜单栏与第一个菜单栏和第三个菜单栏中间插入了一个分割线，要插入分割线很简单，只要将一个 Separator Menu Item 拖到你的 Menu 中就可以了。

我给第一个菜单栏添加了一个可以显示图片的视图。要想实现这个也很简单。你可以在右侧下面的 Object library 中将一个 Custom View 拖到 xib 中。然后在里面添加一个 Image View，并给 Image View 设置一个图片。就像下图这样:

![](http://othizsxsl.bkt.clouddn.com/macapp07.png)

现在选中第一个菜单栏 (Item 1)，按住 `control` 键，拖动鼠标，在弹出框中选择 view，就像下图中的样子。

![](http://othizsxsl.bkt.clouddn.com/macapp08.png)

我们创建了自己的菜单，现在只需要将菜单与 Status Item 联系起来。这个步骤同样很简单。将你的 Menu 创建一个 Outlet ，拖到 AppDelegate 中：

```swift
 @IBOutlet weak var menu: NSMenu!
```

然后在 `applicationDidFinishLaunching ` 中添加下面一行代码:

```swift
 func applicationDidFinishLaunching(_ aNotification: Notification) {
     ...
     
     statusItem.menu = menu
 }
```

运行你的应用，点击系统菜单栏中你的应用的图标，果然出现了刚刚在 xib 中创建的 Menu。现在你的 Status Bar 应用终于有模有样了。接着我们可以先实现菜单中的 Quit 功能。

在你的 xib 中选中 Quit 这个菜单栏，按住 `control` 键，拖动鼠标，在你的 AppDelegate 中创建一个 Action，然后在你的方法中添加如下代码：

```swift
 @IBAction func quit(_ sender: NSMenuItem) {
     NSApp.terminate(nil)
 }
```

你可以运行你的应用看看 Quit 是不是可以起效，不出意外，点击 Quit，你的应用就能退出了，同时快捷键也可以让你的应用退出了。

下图中就是应用最终的样子:

![](http://othizsxsl.bkt.clouddn.com/macapp00.png)



# 总结

通过上面简单的几个步骤，我们很快就实现了自己的第一个 Status Bar 应用。为了让新手也能上手，所以文章写的比较啰嗦，如果你已经有过 iOS 的开发经验，那么你可以跳着看，完成这个简单的  Status Bar 应用可能都不会花费你5分钟时间。

最后总结一下创建我们的 Status Bar 应用的几个关键步骤：

* 创建并设置 Status Item
* 隐藏系统提供的默认窗口
* 从 Dock 栏中隐藏应用图标
* 给 Status Item 添加 Menu
* 实现 Menu Item 的功能（退出功能）

后面我会教你如何实现点击菜单栏弹出一个新的窗口。就像我们的应用中，点击 Preferences，会弹出一个偏好设置的窗口。你不用担心这会很难，这比你想象的要简单的多。

