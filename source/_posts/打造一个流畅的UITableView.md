---
title: 打造一个流畅的UITableView
date: 2016-06-23 09:16:16
tags: iOS
category: iOS
---
Table view需要有很好的滚动性能，不然用户会在滚动过程中发现动画的瑕疵。
为了保证table view平滑滚动，确保你采取了以下的措施:

<!--more-->

* 正确使用`reuseIdentifier`来重用cell
* 尽量使所有的view opaque，包括cell自身
* 避免图片缩放
* 缓存行高
* 尽量不要在`cellForRowAtIndexPath:`中设置数据，如果你需要用到它，只用一次然后缓存结果
* 对齐像素
* 使用`rowHeight`, `sectionFooterHeight` 和 `sectionHeaderHeight`来设定固定的高，不要请求delegate



## 1. 正确使用 **reuseIdentifier** 来重用cell
一个开发中常见的错误就是没有给UITableViewCells， UICollectionViewCells，甚至是UITableViewHeaderFooterViews设置正确的reuseIdentifier。

为了性能最优化，table view用 `tableView:cellForRowAtIndexPath:` 为rows分配cells的时候，它的数据应该重用自UITableViewCell。

不使用reuseIdentifier的话，每显示一行，table view就不得不创建全新的cell。这对性能的影响可是相当大的，尤其会使app的滚动体验大打折扣。所以在使用 UITableViewCell， UICollectionViewCell，或者 UITableViewHeaderFooterView 的时候一定要使用reuseIdentifier。
```
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
  static NSString *ID = @"cell";
  TableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:ID forIndexPath:indexPath];
  
  return cell;
}

- (UIView *)tableView:(UITableView *)tableView viewForFooterInSection:(NSInteger)section {
  static NSString *footerViewWithIdentifie = @"footer";
  UITableViewHeaderFooterView *footer = [tableView dequeueReusableHeaderFooterViewWithIdentifier:footerViewWithIdentifie];
  return footer;
}
```

## 2. 尽量把views设置为不透明
如果你有不透明的Views，你应该设置它们的opaque属性为YES。一些没有被设置为opaque的视图，因为透明通道的存在，系统需要去计算图层堆叠后像素点的真实颜色，这就会产生是混合(blending)操作。我们可以通过模拟器的Debug\Color Blended Layers 选项来查看哪些 view 没有设置为不透明。选中’Color Blended Layers‘。然后iOS模拟器就会将全部区域显示为两种颜色：绿色和红色。绿色区域表示没有混合，但红色区域表示有混合操作。

![opaque](opaque.png)

如果屏幕是静止的，那么这个opaque属性的设置与否不是一个大问题。但是，如果 view 是嵌入到 scroll view 中的，或者是复杂动画的一部分，不将设置这个属性的话肯定会影响程序的性能。所以为了程序的性能，尽可能的将view设置为不透明。

## 3. 避免图片缩放
如果要在 `UIImageView` 中显示一个来自bundle的图片，你应保证图片的大小和 `UIImageView` 的大小相同。在运行中缩放图片是很耗费资源的，特别是 `UIImageView` 嵌套在 `UIScrollView` 中的情况下。如果不做任何处理，直接将图片丢进去，问题就大了，这意味着，GPU需要对大图进行缩放到小的区域显示，需要做像素点的sampling，这种smapling的代价很高，又需要兼顾pixel alignment。计算量会飙升。
如果图片是从远端服务加载的你不能控制图片大小，比如在下载前调整到合适大小的话，你可以在下载完成后，最好是用background thread，缩放一次，然后在UIImageView中使用缩放后的图片。
```
- (UIImage *)imageByScalingAndCroppingForSize:(CGSize)targetSize
{
    UIImage *sourceImage = self;
    UIImage *newImage = nil;
    CGSize imageSize = sourceImage.size;
    CGFloat width = imageSize.width;
    CGFloat height = imageSize.height;
    CGFloat targetWidth = targetSize.width;
    CGFloat targetHeight = targetSize.height;
    CGFloat scaleFactor = 0.0;
    CGFloat scaledWidth = targetWidth;
    CGFloat scaledHeight = targetHeight;
    CGPoint thumbnailPoint = CGPointMake(0.0,0.0);
    
    if (CGSizeEqualToSize(imageSize, targetSize) == NO) {
        CGFloat widthFactor = targetWidth / width;
        CGFloat heightFactor = targetHeight / height;
        
        if (widthFactor > heightFactor)
            scaleFactor = widthFactor; // scale to fit height
        else
            scaleFactor = heightFactor; // scale to fit width
        scaledWidth  = width * scaleFactor;
        scaledHeight = height * scaleFactor;
        
        // center the image
        if (widthFactor > heightFactor) {
            thumbnailPoint.y = (targetHeight - scaledHeight) * 0.5;
        }
        else if (widthFactor < heightFactor) {
            thumbnailPoint.x = (targetWidth - scaledWidth) * 0.5;
        }
    }
    
    UIGraphicsBeginImageContext(targetSize); // this will crop
    
    CGRect thumbnailRect = CGRectZero;
    thumbnailRect.origin = thumbnailPoint;
    thumbnailRect.size.width  = scaledWidth;
    thumbnailRect.size.height = scaledHeight;
    
    [sourceImage drawInRect:thumbnailRect];
    
    newImage = UIGraphicsGetImageFromCurrentImageContext();
    if(newImage == nil)
        NSLog(@"could not scale image");
  
    UIGraphicsEndImageContext();
  
    return newImage;
}
```

## 4. 缓存行高
这个方法对于cell定高的UITableView来说没有意义，但如果由于某些原因需要动态高度的cell的话，这个方法可以很容易地让滑动更流畅。

UITableView的delegate方法`tableView:heightForRowAtIndexPath:`会为每个cell调用一次，所以你应该非常快地返回高度值，避免做一些复杂的高度计算。所以如果你需要动态计算cell的高度的话，应该在调用这个方法之前就计算好高度，并将其缓存起来。我的习惯是在从服务器获取完数据后，在做数据模型化的时候计算内容的高度，并用属性保存起来。

```
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
  CellModel *model = self.cellModels[indexPath.row];
  return model.rowHeight;
}
```

如果你的cell高度是固定的话，请使用`rowHeight`, `sectionFooterHeight` 和 `sectionHeaderHeight`来设定固定的高，不要请求delegate。因为`tableView:heightForRowAtIndexPath:`会为每个cell调用一次。

## 5. 尽量不要在`cellForRowAtIndexPath:`中设置数据
在UITableView的dataSource中实现的`tableView:cellForRowAtIndexPath:`方法，需要为每个cell调用一次，它应该快速执行。所以你需要尽可能快地返回重用cell实例。不要在这里去执行数据绑定，因为目前在屏幕上还没有cell。为了执行数据绑定，可以在UITableView的delegate方法`tableView:willDisplayCell:forRowAtIndexPath:`中进行。这个方法在显示cell之前会被调用。

## 6. 对齐像素
在完美的世界中(我们尝试构建的)，屏幕点总是被处理成物理像素的整型坐标。但在现实生活中它可能是浮点值，例如，线段可能起始于x为0.25的地方。这时候，iOS将执行子像素渲染。

这一技术在应用于特定类型的内容(如文本)时很有意义。但当我们绘制平滑直线时则没有必要。
如果所有的平滑线段都使用子像素渲染技术来渲染，那你会让iOS执行一些不必要的任务，从而降低FPS。

什么情况下会出现这种不必要的子像素抗锯齿操作呢？最常发生的情况是通过代码计算而变成浮点值的视图坐标，或者是一些不正确的图片资源，这些图片的大小不是对齐到屏幕的物理像素上的（例如，你有一张在Retina显示屏上的大小为60X61的图片，而不是60X60的）。

我们可以在iOS模拟器上运行程序，在”Debug“菜单中选中”Color Misaligned Image“。
这一次有两种高亮区域：品红色区域会执行子像素渲染，而黄色区域是图片大小没有对齐的情况。

![](yellow.png)

所以为了避免出现上面的情况，要做到这两点：

* 对所有像素相关的数据做四舍五入处理（使用ceilf, floorf和CGRectIntegral），包括点坐标，UIView的高度和宽度。
* 跟踪你的图像资源：图片必须是像素完美的，否则在Retina屏幕上渲染时，它会做不必要的抗锯齿处理。

## 7. 少用masksToBounds
日常生产中app布局离不开美丽的圆角(RounderCorner)，特别是用圆角UIImageView来做数据呈现交互，但是这种柔和易于让人接受的视图效果并不仅仅是改变了一个形状那么简单，需要付出一定的性能代价。
相信这已经是总所周知的问题了，日常我们使用layer的两个属性，简单的两行代码就能实现圆角的呈现

```
imageView.layer.cornerRadius = 20;
imageView.layer.masksToBounds = YES;
```

由于这样处理的渲染机制是GPU在当前屏幕缓冲区外新开辟一个渲染缓冲区进行工作，也就是离屏渲染，这会给我们带来额外的性能损耗，如果这样的圆角操作达到一定数量，会触发缓冲区和上下文的的频繁切换，这个才是最致命的，创建新的缓冲区代价都不算大，付出最大代价的是上下文切换。性能的代价会宏观地表现在用户体验上----掉帧。

如果你非得使用`cornerRadius`呢？如果你非得这做的话，那么这样也可以拯救你：

```
self.layer.shouldRasterize = YES;
self.layer.rasterizationScale = [UIScreen mainScreen].scale;
```

`shouldRasterize = YES` 会使视图渲染内容被缓存起来，下次绘制的时候可以直接显示缓存，当然要在视图内容不改变的情况下。

最好的方式是：预先生成圆角图片，并缓存起来。预处理圆角图片可以在后台处理，处理完毕后缓存起来，再在主线程显示，这就避免了不必要的离屏渲染了。

```
@implementation UIImageView (CornerRadius)

- (void)hu_setCornerRadius:(CGFloat)radius {
  
  dispatch_queue_t bq = dispatch_queue_create("com.hujewelz.cornerradius", DISPATCH_QUEUE_CONCURRENT);
  dispatch_async(bq, ^{
    
    UIGraphicsBeginImageContextWithOptions(self.bounds.size, NO, [UIScreen mainScreen].scale);
    
    UIBezierPath *path = [UIBezierPath bezierPathWithRoundedRect:self.bounds byRoundingCorners:UIRectCornerAllCorners cornerRadii:CGSizeMake(radius, radius)];
    [path addClip];
    
    [self.image drawInRect:self.bounds];
    
    UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
   
    dispatch_async(dispatch_get_main_queue(), ^{
      self.image = image;
    });
    
    UIGraphicsEndImageContext();
    
  });
  
  
}
```

我们可以在iOS模拟器上运行程序，在”Debug“菜单中选中”Color Offscreen-Rendered“。
黄色区域表示产生了离屏渲染。


