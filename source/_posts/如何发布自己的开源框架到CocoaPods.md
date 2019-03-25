---
title: 如何发布自己的开源框架到CocoaPods
date: 2016-02-14 08:52:00
tags: 
  - iOS 
  - CocoaPods
category: iOS
thumbnailImage: https://www.raywenderlich.com/wp-content/uploads/2015/02/cocoapods_logo-250x250.png
autoThumbnailImage: yes
coverSize: partial
coverImage: http://ashishkakkad.com/wp-content/uploads/2016/01/CocoaPodsLogo.png
---
在开发过程中，经常会使用到第三框架，我们通过一个`pod install`命令，很方便的就将第三方框架加到我们自己的项目中。  
如果我们也想将自己写的组件或库开源出去，让别人也可以通过`pod install`命令安装自己的框架该怎么做呢？
<!--more-->
下面，我就教大家一步一步的将自己的pods发布到`CocoaPods` 中。如果你现在对`CocoaPods`还不太了解，推荐你看一看这篇文章：[用CocoaPods做iOS程序的依赖管理](http://blog.devtang.com/2014/05/25/use-cocoapod-to-manage-ios-lib-dependency/)

## 创建自己项目的Podspec描述文件
下面我会通过一个名为`HUPhotoBrowser`的项目来讲解一下整个过程。
项目发布到`github`后，需要打上`tag`。之后我们在工程根目录中初始化一个Podspec文件：
```
pod spec create HUPhotoBrowser
```
该命令将在本目录产生一个名为`HUPhotoBrowser.podspec`文件。用编辑器打开该文件，里面已经有非常丰富的说明文档。下面介绍如何声明第三方库的代码目录和资源目录，还有该第三方库所依赖ios核心框架和第三方库。这是我的podspec文件：
```
Pod::Spec.new do |s|
  s.name         = "HUPhotoBrowser"
  s.version      = "0.0.2"
  s.summary      = "photo browser for ios."
  s.homepage     = "https://github.com/hujewelz/HUPhotoBrowser"
  s.license      = "MIT"
  s.author             = { "Jewelz Hu" => "hujewelz@163.com" }
  s.platform     = :ios, "7.0"
  s.source       = { :git => "https://github.com/hujewelz/HUPhotoBrowser.git", :tag => "0.0.2" }
  s.source_files  = "HUPhotoBrowser", "HUPhotoBrowser/**/*.{h,m}"
   s.framework  = "UIKit"
  # s.frameworks = "SomeFramework", "AnotherFramework"
```
`s.name`是我们库的名称，`s.version`是库原代码版本号，`s.summary`是对我们库的一个简单的介绍，`s.homepage`声明库的主页，`s.license`是所采用的授权版本，`s.author`是库的作者。` s.platform`是我们库所支持的软件平台，这在我们最后提交进行编译 时有用。`s.source`声明原代码的地址。我这里是托管在github上,所以这里将地址copy过来就行了。

![屏幕快照 2016-02-26 下午3.02.58.png](http://upload-images.jianshu.io/upload_images/1351863-5f185444531af1d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对于很多第三方库而言，在发布的时候都会打上一个`tag`，如版本0.0.1就会打上一个名为`0.0.1`的`tag`,你也可以选择一个最新的`commit`来作为该库0.0.1版的代码, 那么最终source就是这样了：
```
{:git => "https://github.com/hujewelz/HUPhotoBrowser.git", :commit => '65584b0e0b08e01f83e66d487180c164b5182409'}
```
我这里还是使用的tag，所以我这里就是这样的：
```
 { :git => "https://github.com/hujewelz/HUPhotoBrowser.git", :tag => "0.0.2" }
```
以后我们的库有新版本时，我们可以修改相应的`version`和`source`。
`s.source_files`声明了我们库的源代码的位置，所以这个地方不能填错了。
先看一下我的目录结构：

![屏幕快照 2016-02-26 下午3.15.07.png](http://upload-images.jianshu.io/upload_images/1351863-98aca18e60fac44a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以工程根目录下的`HUPhotoBrowse`文件夹才是库的原代码目录。
```
s.source_files  = "HUPhotoBrowser", "HUPhotoBrowser/**/*.{h,m}"
```
目录的层级关系一定要跟代码库的保持一致。这里前一部分可以不用的，因为我这里后一部分的`HUPhotoBrowser/**`与前面是一致的，这个指定的目录下的文件都会进行编译。如果该目录下还有一些资源文件（如图片等），这些文件并不需要进行编译。可以使用`s.resourcs`声明。` *.{h,m}`是一个类似正则表达式的字符串，表示匹配所有以`.h`和`.m`为扩展名的文件。
`s.framework`声明了所依赖的核心库，我这里只用到了`UIKit`,所以是这样的：
``` 
s.framework  = "UIKit"
```
如果你的项目中依赖多个库，可以使用
```
s.frameworks = "SomeFramework", "AnotherFramework"
```
当然，我们开发的库中也可能还依赖第三方库，例如`JSONKit`，那么，就可以做如下声明:
```
 s.dependency "JSONKit", "~> 1.4"
```
如果有多个需要填写多个`s.dependency`。  
编辑完`podspec`文件后，需要验证一下这个文件是否可用，如果有任何WARNING或者ERROR都是不可以的，它就不能被添加到Spec Repo中，不过xcode的WARNING是可以存在的，验证需要执行命令：
```
pod spec lint PodName.podspec
```
当看到`HUPhotoBrowser passed validation.`时，说明验证通过了。在检测你的podspec时候，如果直接用pod spec lint xxx.podspec的话，出现错误它只会直接一句红色的话`The spec did not pass validation, due to 1 error.`告诉你的有多少个error和warning，而不会具体的指出你的错误出在哪里，这时候你可以在这句指令后面加上参数--verbose 这样就会告诉你具体的错误信息。这样根据它提示你的错误信息去解决就可以了。

编辑好`podspec`文件后就可以将该`podspec`文件保存到本机的`~/.cocoapods/repos/master/Specs`目录中仅供自己使用，也可以将其提交到CocoaPods/Specs代码库中。下面我们先将其保存到本机中：

![屏幕快照 2016-02-26 下午3.44.31.png](http://upload-images.jianshu.io/upload_images/1351863-c8e31c301e9c2c59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下面可以看一下是否可以通过搜索找到该库:

![屏幕快照 2016-02-26 下午3.48.06.png](http://upload-images.jianshu.io/upload_images/1351863-51c1e65c0c2a5a9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
同样在需要依赖于`HUPhotoBrowser`这个库的项目，可以将下列添加到项目的`Podfile`文件中
```
pod 'HUPhotoBrowser', '~0.0.2'
```
保存文件，并用`pod install`安装`HUPhotoBrowser`库。

通过以上步骤创建Pod库还只能供自己使用，下面会继续讲解如何将其提交到CocoaPods/Specs代码库中，让其他人也可以通过`pod install`安装我们的开源库。

## CocoaPods Trunk发布自己的Pods
在cocoapods使用了trunk之后，`CocoaPods` 需要0.33以上版本，用 `pod --version`查看版本，如果版本低，需要更新。
### 注册Trunk
```
$ pod trunk register orta@cocoapods.org 'Orta Therox' --description='macbook air'
```
大家在注册时需要替换成自己的邮箱和用户名，一切顺利的话就会受到一份邮件，点击邮件中的链接后验证一下：
```
pod trunk me
```

![屏幕快照 2016-02-26 下午4.05.42.png](http://upload-images.jianshu.io/upload_images/1351863-641b06a41444a0dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当然，如果你的pod是由多人维护的，你也可以添加其他维护者:
```
$ pod trunk add-owner ARAnalytics kyle@cocoapods.org
```
上面的工作完成之后，我们就可以开始 `trunk push`了。
### Trunk push
`pod trunk push` 命令会首先验证你本地的`podspec`文件(是否有错误)，之后会上传`spec`文件到`trunk`，最后会将你上传的`podspec`文件转换为需要的`json`文件。在工程根目录(包含有.podspec)下执行命令：
```
pod trunk push
```
如果在`trunk push`过程中报错了，仔细查看一下错误信息。我当初就是使用了`podspec`文件中描述的版本所没有的API，之后修改`podspec`文件中` s.platform = :ios, "7.0"`就可以了。

![屏幕快照 2016-02-26 下午4.12.59.png](http://upload-images.jianshu.io/upload_images/1351863-70f2bc73825180bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果你能看的上面的结果说明上传成功了。我们也可以在本地的`~/.cocoapods/repos/master/Specs`目录下看到转换之后的`json`文件,

![屏幕快照 2016-02-26 下午4.16.56.png](http://upload-images.jianshu.io/upload_images/1351863-9f93e6c957de080f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
至此我们整个制作自己的开源库的过程就完成了，以后有新版本只需要修改工程根目录下的`podspec`文件就行了，然后重新执行`pod trunk push`命令。

## 最后
最后对这个过程做个总结：
1. 开源库发布之后，需要打上`tag`
2. 进入到项目根目录下，创建`podspec`文件
```
pod spec create PodName
```
3. 编辑`podspec`文件中的相关信息，有两个比较重要的地方` s.source`和` s.source_files `,可以验证是否有误：
```
pod spec lint PodName.podspec
```
4. 注册pod trunk
```
$ pod trunk register orta@cocoapods.org 'Orta Therox' --description='macbook air'
```
5. 发布到pod trunk
```
pod trunk push [NAME.podspec]
```
该命令在包含有`.podspec`文件的目录下执行
6. 更新pod库
```
pod setup
```
如果`pod trunk push`成功后无法`pod search`到自己的库，可执行该命令。

## 最后的最后
哈哈。好吧，我承认其实我是来打广告的。例子中的[**HUPhotoBrowser**](https://github.com/hujewelz/HUPhotoBrowser)是我开源的一个图片浏览器的库，使用起来非常简单，一行代码就以实现图片浏览功能，支持本地和网络图片。希望大家可以支持一下，欢迎大家star。如果有什么问题的话可以直接issue我。最后，希望能跟大家共同进步。项目地址：[HUPhotoBrowser](https://github.com/hujewelz/HUPhotoBrowser)

