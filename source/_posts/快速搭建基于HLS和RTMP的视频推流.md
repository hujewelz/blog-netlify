---
title: 快速搭建基于HLS和RTMP的视频推流
date: 2016-11-22 10:04:49
tags: 直播
---
在视频直播越来越火热的今天，作为一个开发者有必要了解一个完整的直播流程是怎样的。在一个完整的手机机直播主要包含了以下几个环节：

<!--more-->

* 推流端：采集，前处理，编码，推流。

* 服务端处理：转码，录制，截图，鉴黄。

* 播放器：拉流，解码，渲染；互动系统：聊天室，礼物系统等。


# 前言
在视频直播越来越火热的今天，作为一个开发者有必要了解一个完整的直播流程是怎样的。在一个完整的手机机直播主要包含了以下几个环节：

* 推流端：采集，前处理，编码，推流。

* 服务端处理：转码，录制，截图，鉴黄。

* 播放器：拉流，解码，渲染；互动系统：聊天室，礼物系统等。

概括起来就是以下几个步骤：

* 采集: iOS 系统因为软硬件种类不多，硬件适配性比较好，所以比较简单。而 Android 端市面上机型众多，要做些机型的适配工作。PC 端是最麻烦的，各种奇葩摄像头驱动。所以现在很多的中小型直播平台，只做 iOS 端的视频直播。

* 前处理: 美颜算法，视频的模糊效果，水印等都是在这个环节做。目前 iOS 端最著名开源框架的毫无疑问就是 GPUImage 。其中内置了一百多种渲染效果，还支持各种脚本自定义。

* 编码: 重难点在于要在分辨率，帧率，码率，GOP 等参数设计上找到最佳平衡点。iOS8 之后，Apple 开放了 VideoToolbox.framework, 可以直接进行硬编解码，这也是为什么现在大多数直播平台最低只支持到iOS8 的原因之一。iOS 端硬件兼容性比较好，可以直接采取硬编码。而 Android 得硬编码又是一大坑。

* 传输: 这块一般都是交给 CDN 服务商。CDN 只提供带宽和服务器之间的传输，发送端和接收端的网络连接抖动缓存还是要自己实现的。目前国内最大的CDN服务商应该是网宿。

* 服务器处理: 需要在服务器做一些流处理工作，让推送上来的流适配各个平台各种不同的协议，比如:RTMP,HLS,FLV...

* 解码和渲染: 也就即音视频的播放。解码毫无疑问也必须要硬解码。iOS 端兼容较好，Android 依然大坑。这块的难点在于音画同步，国内比较好的开源项目应该是B站开源的[ijkplayer](https://github.com/Bilibili/ijkplayer)。

对于移动端开发者来说，最主要工作的就是推流端和播放器端了。
在本篇文章中我把注意力放在大家不太关注的服务端。同学们只要按照下面的步骤，就能很快的搭建一个视频推流服务器。

# 在 Mac下 搭建 nginx+rtmp 服务器

### 1. 使用 [homebrew](https://brew.sh) 安装 nginx

先 clone nginx 到本地：
```shell
brew tap homebrew/nginx
```
执行安装：
```shell
brew install nginx-full --with-rtmp-module
```
此时, nginx和rtmp模块就安装好了, 输入命令：
```shell
nginx
```
在浏览器里打开 [http://localhost:8080](http://localhost:8080)，如果出现以下页面说明 nginx 安装成功。

![](http://image18-c.poco.cn/mypoco/myphoto/20170322/12/18436043320170322125539085.png?526x218_130)

### 2. 配置 hls 和 rtmp
使用编辑器打开 `usr/local/etc/nginx/nginx.conf`文件，找到 `http` 下的 `server`, 在花括号中添加如下内容：
```
server {
   listen       8080;
   server_name  localhost;

   #charset koi8-r;

   #access_log  logs/host.access.log  main;

   location / {
       root   html;
       index  index.html index.htm;
   }

   # HLS config
   location /hls {
       types {
           application/vnd.apple.mpegurl m3u8;
           video/mp2t ts;
     }
       root html;
       add_header Cache-Control no-cache;
	}
	...
```
这样 HLS 就配置好了。
    
现在来配置 RTMP ，直接滚动到最后一行，在 `http` 的结束(最后的`}`)后面添加如下内容：
```
rtmp {
    server {
       listen 1935;
       application gzhm {
           live on;
           record off;
       }
       #增加对HLS支持开始
       application hls {
           live on;
           hls on;
           hls_path /usr/local/var/www/hls;
       }
   }
}
```
保存配置文件，重新加载 nginx 配置：
```shell
nginx -s reload
```

# 测试推流
我们可以使用 [ffmpeg](http://ffmpeg.org) 来进行推流。关于 ffmpeg 的详细用法，大家可以参考官方文档。 
> FFmpeg是一套可以用来记录、转换数字音频、视频，并能将其转化为流的开源计算机程序。

### 1. 使用 homebrew 安装 ffmpeg
在终端执行命令：

```shell
brew install ffmpeg
```

安装 ffmpeg 时间就要长一点了, 请耐心等待一会。安装完成后，就可以使用 ffmpeg 来推流了。

### 2. 推流
我们可以在终端输入以下命令来推流：
```shell
ffmpeg -loglevel verbose -re -i  视频的全路径  -vcodec libx264 -vprofile baseline -acodec libmp3lame -ar 44100 -ac 1 -f flv rtmp://localhost:1935/hls/文件名称(不包含后缀)
```

然后你就可以在 `/usr/local/var/www/hls` 目录下看到一个后缀名为 `.m3u8` 的文件和 `.ts` 分片文件。
    
这里的输出目录 `/hls` 是在 `nginx.conf`中 配置好的。这里要对应，不然是无法推流成功的。
### 3. 测试拉流
1. HLS 拉流测试：
  
    你可以在电脑 Safari 里输入地址查看视频，也可以用 iPad 或者 iPhone 上的 Safari 来访问（需要把 localhost 改为 nginx 的所在电脑的ip地址）

    ```
    http://localhost:8080/hls/a.m3u8
    ```
    
2. RTMP 拉流测试
    由于浏览器并不支持 rtmp 协议，所以我们需要下载支持 rtmp 协议的视频播放器，可以使用 VLC 播放器。
        
    将视频推流到服务器后，打开 VLC，然后 File -> open network -> 输入：
    ```
    trmp://localhost:1935/hls/a
    ```
    
通过以上简单的几个步骤，就能在自己的电脑上搭建推流服务器了。不过在真实的项目中还是建议使用第三方的直播服务，比如阿里云和腾讯云，他们对视频直播都有比较完整的方案。
    

