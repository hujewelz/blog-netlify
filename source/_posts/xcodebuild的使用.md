---
title: xcodebuild的使用
date: 2017-01-25 18:00:00
tags: Xcode
coverImage: cover.jpg
thumbnailImage: thumbnail.jpg
thumbnailImagePosition: right
---

**xcodebuild** 用于构建 Xcode 项目中包含的一个或多个**target** ，或者构建一个包含在 Xcode 工作区或 Xcode 项目的 **scheme**

<!--more-->

```
xcodebuild [-project projectname] [-target targetname ...] 
           [-configuration 	configurationname]
           [-sdk [sdkfullpath | sdkname]] [buildaction ...] 
           [setting=value ...] [-userdefault=value ...]
           
xcodebuild [-project projectname] -scheme schemename 
           [-destination destinationspecifier]
           [-destination-timeout value] [-configuration configurationname]
           [-sdk [sdkfullpath | sdkname]] [buildaction ...] 
           [setting=value ...] [-userdefault=value ...]
           
xcodebuild -workspace workspacename -scheme schemename 
           [-destination destinationspecifier]
           [-destination-timeout value] [-configuration configurationname]
           [-sdk [sdkfullpath | sdkname]] [buildaction ...] 
           [setting=value ...] [-userdefault=value ...]
           
xcodebuild -version [-sdk [sdkfullpath | sdkname]] [infoitem]
xcodebuild -showsdks
xcodebuild -list [-project projectname | -workspace workspacename]

xcodebuild -exportArchive -exportFormat format 
           -archivePath xcarchivepath 
           -exportPath destinationpath 
           [-exportProvisioningProfile profilename] 
           [-exportSigningIdentity identityname] 
           [-exportInstallerIdentity identityname]
```



#### 使用

要构建一个Xcode项目，请从包含项目的目录运行 **xcodebuild**（即包含 projectname.xcodeproj 的目录）。如果你在这目录中有多个项目，需要使用 **-project** 来指明应该构建哪个项目。默认情况下， **xcodebuild** 会使用默认的构建配置构建项目中列出的第一个 target。 target 的顺序是项目的一个属性，对于项目的所有用户都是一样的。

要构建 Xcode 工作区，你必须同时传递 **-workspace** 和 **-scheme** 选项来定义构建。**scheme** 的参数将控制构建哪些 target 以及它们是如何构建的。不过你仍然可以将其他选项传递给 **xcodebuild** 来覆盖该 scheme 的一些参数。

还有几个参数可以显示有关已安装的Xcode版本或本地目录中项目或工作区的信息，但不启动构建。包括 **-version**, **-showsdks** 和 **-usage**。


#### 选项(options)
* **-project** projectname
	构建由 projectname 指定的项目。如果在同一个目录下有多个项目文件，该选项是必需的。
	
* **-target** targetname
	构建由 targetname 指定的目标。
* **-alltargets**
	构建指定项目中的所有目标
* **-workspace** workspacename
	构建由workspacename指定的工作空间。
* **-scheme** schemename
	构建由 schemename 指定的方案。如果构建一个 workspace ，则为必需。
* **-destination** destinationspecifier
	使用由 destinationpecifier 描述的目标设备。默认为与选定scheme兼容的 destination。
* **-destination-timeout** timeout
	在搜索目标设备时使用的超时时间。默认值是30秒。
* **-configuration** configurationname
	在构建每个 target 时使用由 configurationname 指定的构建配置。
* **-arch** architecture
	构建每个 target 时使用的 architecture。
* **-sdk** [<sdkfullpath> | <sdkname>]
	使用适合于该 SDK 的构建工具，针对指定的 SDK 构建 Xcode 项目或工作区。参数可以是 SDK 的绝对路径，也可以是 SDK 的名称。
* **-showsdks**
	列出 Xcode 中所有可用的 SDK，包括适合与 `-sdk` 一起使用的规范名称。不会启动构建。
* **-list**
	列出项目中的所有 target 和配置，或工作区中的 scheme。不会启动构建。
* **-derivedDataPath** path
	在执行构建操作时覆盖用于 derived data 的文件夹。
* **-resultBundlePath** path
	将包绑定到指定的路径，并在其中的方案上执行构建操作。
* **-exportArchive**
	指定应导出归档。 需要`-exportFormat`，`-archivePath`和`-exportPath`。 不能与构建操作一起使用。
* **-exportFormat** format
	指定归档应该导出到的格式。有效的格式是IPA（iOS），PKG（Mac）和APP。 如果未指定xcodebuild 将尝试自动检测格式为IPA或PKG。
* **-archivePath** xcarchivepath
	指定归档操作生成的归档的路径，或者指定归档在`-exportArchive`时被导出。
* **-exportPath** destinationpath
	指定导出产品的目的路径，包括导出文件的名称。
* **-exportProvisioningProfile** profilename
	指定导出归档时使用的 provisioning profile 文件。
* **-exportSigningIdentity** identityname
	指定导出归档时使用的应用程序签名标识。如果可能的话，这可以从`-exportProvisioningProfile`中推断出来。
* **-exportInstallerIdentity** identityname
	指定导出归档时使用的安装程序签名标识。、也可以从`-exportSigningIdentity`或`-exportProvisioningProfile`中推断出来。
* **-exportWithOriginalSigningIdentity**
	指定在导出文件时使用的用于创建归档的签名标识。

##### buildaction ...
指定要在 target 上执行的构建操作（或多个操作）:

action| 描述
----- | --------
build | 在构建根目录（SYMROOT）中构建目标。这是默认的构建操作。
analyze|从构建根目录（SYMROOT）构建和分析target或scheme。需要指定一个scheme。
archive|从构建根目录（SYMROOT）归档一个scheme。需要指定一个scheme。
test | 从构建根目录（SYMROOT）测试一个scheme。这需要指定一个scheme和一个目的地。
installsrc|将项目源复制到源根目录（SRCROOT）。
install | 构建target并将其安装到 DSTROOT 中的目标安装目录中。
clean | 从构建根目录（SYMROOT）中删除构建产品和中间文件。

##### Destinations
**-destination**选项采用描述设备（或多个设备）作为目的地的目标说明符作为其参数。目标说明符是由一组逗号分隔的键值对组成的单个参数。 可以多次指定**-destination**选项，以使**xcodebuild**在多个目标上执行指定的操作。

目标说明符可能包含平台密钥以指定其中一个受支持的目标平台。 应根据您选择的设备的平台提供附加的键

某些设备可能需要一些时间才能查找。 **-destination-timeout** 选项可用于指定在设备被认为不可用之前等待的时间。默认为30秒。

目前，xcodebuild支持这些平台：
```
OS X             本地 Mac，在Xcode界面中称为My Mac，支持以下内容: 
                 arch 要使用的体系结构，x86_64（默认）或者i386。
                 
iOS              iOS 设备，支持以下内容：
                 name 使用的设备的名称
                 id   在Xcode Organizer中“设备”选项卡中的设备的标识符。
                 
iOS Simulator    iOS 模拟器，支持以下内容: 
                 name 在 Xcode 的界面中提供的模拟器的全名。
                 OS   要模拟的iOS版本（如6._）或最新字符串（默认值），以指示该版本Xcode   
                 支持的最新iOS版本。
```
e.g.
```shell
$ xcodebuild -workspace MyWorkspace.xcworkspace -scheme MyScheme -destination 'platform=OS X,arch=x86_64' build
```


##### 导出归档
**-exportArchive** 选项指定 **xcodebuild** 应该导出由 **-archivePath** 指定的归档并转换为 **-exportFormat** 指定的格式。导出的产品将放置在由 **-exportPath** 指定的路径中。 导出归档时重新签名是可选的。 配置文件(Provisioning Profile)可以由 **-exportProvisioningProfile** 指定。在某些情况下，应该在导出期间使用的应用程序签名标识可以从配置文件中确定。对于不可能的情况（包括在导出产品中没有嵌入配置文件的情况），可以使用 **-exportSigningIdentity** 指定应用程序签名标识。将 Mac 归档文件导出为 PKG 时，可以使用安装程序签名标识对导出的包进行签名。 这可以从应用程序签名标识中推断出来（例如，如果为应用程序签名标识指定了“Developer ID Application”，则将自动推断“Developer ID Installer”），也可以使用**-exportInstallerIdentity**明确指定它。


#### 例子
```shell
$ xcodebuild clean install
```
清理构建目录; 然后在 **xcodebuild** 启动的目录中构建并安装 Xcode 项目中的第一个target。

-----

```shell
$ xcodebuild -target MyTarget OBJROOT=/Build/MyProj/Obj.root SYMROOT=/Build/MyProj/Sym.root
```
在 **xcodebuild** 开始的目录中的 Xcode 项目中构建 MyTarget，将中间文件放入`/Build/MyProj/Obj.root`目录中，并将构建的产品放入`/Build/MyProj/Sym.root`目录中。

-----

```shell
$ xcodebuild -sdk macosx10.6
```
在 Mac OS X 10.6 SDK 中启动 **xcodebuild** 的目录中生成 Xcode 项目。可以使用**-showsdks**选项查看所有可用SDK的名称。

-----

```shell
$ xcodebuild -workspace MyWorkspace.xcworkspace -scheme MyScheme
```
在 Xcode 工作区 MyWorkspace.xcworkspace 中构建方案 MyScheme。

-------

```shell
$ xcodebuild -workspace MyWorkspace.xcworkspace -scheme MyScheme archive
```
在 Xcode 工作区 MyWorkspace.xcworkspace 中归档方案 MyScheme。

---------

```shell
$ xcodebuild -workspace MyWorkspace.xcworkspace -scheme MyScheme -destination 'platform=OS X,arch=x86_64' test
```
在 Xcode 工作区 MyWorkspace.xcworkspace 中使用Xcode中描述为'platform=OS X,arch=x86_64'的目标测试 MyScheme 中的方案。


```shell
$ xcodebuild -workspace MyWorkspace.xcworkspace -scheme MyScheme -destination 'platform=iOS Simulator,name=iPhone' -destination 'platform=iOS,name=My iPad' test
```
-------

```shell
$ xcodebuild -workspace MyWorkspace.xcworkspace -scheme MyScheme -destination generic/platform=iOS build
```
使用通用 iOS 设备在 Xcode 工作区 MyWorkspace.xcworkspace 中构建方案 MyScheme。

------

```shell
$ xcodebuild -exportArchive -exportFormat IPA -archivePath MyMobileApp.xcarchive -exportPath MyMobileApp.ipa -exportProvisioningProfile 'MyMobileApp Distribution Profile'
```
使用 "MyMobileApp Distribution Profile" 配置文件将归档 MyMobileApp.xcarchive 作为 IPA 文件导出到MyMobileApp.ipa目录中。

------

```shell
$ xcodebuild -exportArchive -exportFormat APP -archivePath MyMacApp.xcarchive -exportPath MyMacApp.pkg -exportSigningIdentity 'Developer ID Application: My Team'
```
使用 "Developer ID Installer：My Team" 签名将 MyMacApp.xcarchive 作为 PKG 文件导出到MyMacApp.pkg目录中。


