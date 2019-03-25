---
title: 使用一个UIImageView实现轮播控件
date: 2016-09-24 19:03:22
tags: Swift
category: Swift
keywords:
- iOS
- swift
- UIImageView
---
在做iOS开发中，图片轮播是一个比较频繁的需求了。网上也有很多比较好的实现，有使用2个、3个`UIImageView`的，也有使用`UICollectionView`的。这里我要讲的是如何用一个`UIImageView`实现一个图片轮播控件，当然加载网络图片是必须的。闲话少说，直接进入正题：
<!--more-->
![](http://upload-images.jianshu.io/upload_images/1351863-f3b399cb6f10893e.gif?imageMogr2/auto-orient/strip)

## 构建UI
在轮播控件中只需要一个`UIImageView`和一个`UIPageControl`即可：

```swift
lazy var imageView: UIImageView = {
    let imageV = UIImageView()
    imageV.userInteractionEnabled = true
    imageV.translatesAutoresizingMaskIntoConstraints = false
    return imageV
}()
 
lazy var pageControl: UIPageControl = {
    let pageC = UIPageControl()
    pageC.currentPage = 0
    pageC.translatesAutoresizingMaskIntoConstraints = false
    return pageC
}()
```
添加好约束即可。然后需要给图片添加一个左划和右划手势，以及一个点击的手势

```swift
private func addGesture() {
    let left = UISwipeGestureRecognizer(target: self, action: #selector(self.swipGesterHandelr(_:)))
    left.direction = .Left
    self.imageView.addGestureRecognizer(left)        
    let right = UISwipeGestureRecognizer(target: self, action: #selector(self.swipGesterHandelr(_:)))
    right.direction = .Right
    self.imageView.addGestureRecognizer(right)
        
    let tap = UITapGestureRecognizer(target: self, action: #selector(self.tapGesterHandelr(_:)))
    self.imageView.addGestureRecognizer(tap)
}
```
## 创建并启动定时器
既然是轮播，那么就必须得有一个定时器吧，当你使用定时器的时候，就一定要注意定时器的销毁。当进入界面时就要启动定时器，当离开界面时就要销毁定时器。我以前使用过别人写的轮播控件，当我已经离开界面进入下一个界面时，定时器竟然还在运行，这样是不对的。其实实现起来很简单：

```swift
public override func willMoveToWindow(newWindow: UIWindow?) {
    super.willMoveToWindow(newWindow)
        
    guard let _ = newWindow else {
        stopTimer()
            return
    }
    refreshTimer()
}

private func stopTimer() {
   timer?.invalidate()
   timer = nil
}

private func refreshTimer() {
    if timer == nil && autoScrollEnable {
        timer = NSTimer.scheduledTimerWithTimeInterval(timeInterval, target: self, selector: #selector(self.timeAction), userInfo: nil, repeats: true)
    }
}
```
## 自动轮播的实现
其实这里才是重点的好吧，一个`UIImageView`要实现轮播效果是很简单的，只需要使用系统的`CATransition`就可以了，来，上代码：

```swift
@objc private func timeAction() {
    scrollWithDirection(.Left)
}

private func scrollWithDirection(direction: ScrollDirection) {
    switch direction {
    case .Left:
        index += 1
        if index > imageCounts - 1 {
            index = 0
        }
    case .Right:
        index -= 1
        if index < 0 {
            index = imageCounts - 1
        }
    }
    
    if images.count > 0 {
        self.imageView.image = images[index]
    }
    else {
        if let url = NSURL(string: imageURLStringGroup[index]) {
            self.imageView.hu_setImageWithURL(url, placeholderImage: placeholderImage)
        }
    }
    
    addScrollAnimationWithDirection(direction)
}

private func addScrollAnimationWithDirection(direction: ScrollDirection) {
    let animation = CATransition()
    animation.duration = 0.4
    animation.type = kCATransitionPush
    
    switch direction {
    case .Left:
        animation.subtype = kCATransitionFromRight
    case .Right:
        animation.subtype = kCATransitionFromLeft
    }
    
    self.imageView.layer.addAnimation(animation, forKey:"scroll")
    self.pageControl.currentPage = index
}
```
代码非常简单，所以就不一一解释了。现在我们已经构建好了UI，创建并启动了定时器（而且能成功销毁），并成功添加了手势，这样这个图片轮播控件就可以正常工作了。但是在我们成功给轮播图设置图片后，我们得做些其他的工作：

```swift
public var images: [UIImage] = [] {
    willSet {
        imageCounts = newValue.count
        imageView.image = newValue.first
        pageControl.numberOfPages = newValue.count
    }
}
    
public var imageURLStringGroup: [String] = [] {
    willSet {
        imageCounts = newValue.count
            
        guard let url = NSURL(string: newValue.first!) else {
            return
        }
        imageView .hu_setImageWithURL(url, placeholderImage: placeholderImage)
        pageControl.numberOfPages = newValue.count
    }
}
```
现在就可以使用了：

```swift
override func viewDidLoad() {
    super.viewDidLoad()
        
    let images = ["a.jpg", "b.jpg","c.jpg","d.jpg",].flatMap {
        return UIImage(named: $0)
    }
        
    let cycleView = HUScrollCycleView(frame: CGRectMake(0, 64, self.view.frame.size.width, 200))
    self.view.addSubview(cycleView)
    cycleView.delegate = self
    cycleView.images = images
    cycleView.currentPageIndicatorTintColor = UIColor.redColor()
//  cycleView.placeholderImage = UIImage(named:"a.jpg")
//  cycleView.imageURLStringGroup = ["http://1.7feel.cc/yungou/statics/uploads/banner/20160715/85964915563838.jpg",
//                                         "http://1.7feel.cc/yungou/statics/uploads/banner/20160715/20274054563730.jpg",
//                                         "http://1.7feel.cc/yungou/statics/uploads/banner/20160715/40912708563719.jpg",
//                                         "http://1.7feel.cc/yungou/statics/uploads/touimg/20160718/img193.jpg"];
    }
```
最后附上[GitHub](https://github.com/hujewelz/HUScrollCycle)地址。[**HUScrollCycle**](https://github.com/hujewelz/HUScrollCycle)中的网络图片下载并没有使用其他第三方库，这里我使用的是以前用`Objective-C`写的`HUWebImageDownloader`，它在我维护的一个图片浏览器第三方库[HUPhotoBrowser](https://github.com/hujewelz/HUPhotoBrowser)中使用的，它支持网络图片和本地相册图片的浏览和多选，有兴趣的童鞋可以[点这里](https://github.com/hujewelz/HUPhotoBrowser)。
所以你完全可以放心使用**HUScrollCycle**而不用担心对你项目产生影响。你只要按照上面的链接中的方法正确的导入`HUScrollCycle-Bridging-Header.h`就可以正常使用了。

