---
title: iOS 应用启动流程分析-dyld
date: 2017-10-11 10:04:03
tags: 原理 dyld
categories: iOS
---

iOS 应用程序启动过程可以以 `main` 函数为界，这里我们先不用管 `main() `函数调用后的过程，主要来分析一下 `mian()` 函数调用之前的dyld阶段。

我们可以先写个简单的程序来看看系统在调用 `main()` 之前，调用了哪些函数。

<!--more-->

![](01.png)

这里给 `load` 方法添加了一个断点。从调用栈可以看到最先调用的是 `__dyld_start` 函数。我们可以从 [dyld 源码](https://github.com/opensource-apple/dyld) dyldStartup.s 中找到 `__dyld_start` 的实现。此函数由汇编实现，兼容各种平台架构，此处主要以arm64 架构下的汇编代码为例：

```asm6502
#if __arm64__
	.data
	.align 3
__dso_static: 
	.quad   ___dso_handle

	.text
	.align 2
	.globl __dyld_start
__dyld_start:
	mov 	x28, sp
	and     sp, x28, #~15		// force 16-byte alignment of stack
	mov	x0, #0
	mov	x1, #0
	stp	x1, x0, [sp, #-16]!	// make aligned terminating frame
	mov	fp, sp			// set up fp to point to terminating frame
	sub	sp, sp, #16             // make room for local variables
	ldr     x0, [x28]		// get app's mh into x0
 	ldr     x1, [x28, #8]           // get argc into x1 (kernel passes 32-bit int argc as 64-bits on stack to keep alignment)
	add     x2, x28, #16		// get argv into x2
	adrp	x4,___dso_handle@page
	add 	x4,x4,___dso_handle@pageoff // get dyld's mh in to x4
	adrp	x3,__dso_static@page
	ldr 	x3,[x3,__dso_static@pageoff] // get unslid start of dyld
	sub 	x3,x4,x3		// x3 now has slide of dyld
	mov	x5,sp                   // x5 has &startGlue
	
	// call dyldbootstrap::start(app_mh, argc, argv, slide, dyld_mh, &startGlue)
	bl	__ZN13dyldbootstrap5startEPK12macho_headeriPPKclS2_Pm
	mov	x16,x0                  // save entry point address in x16
	ldr     x1, [sp]
	cmp	x1, #0
	b.ne	Lnew

	// LC_UNIXTHREAD way, clean up stack and jump to result
	add	sp, x28, #8		// restore unaligned stack pointer without app mh
	br	x16			// jump to the program's entry point

	// LC_MAIN case, set up stack for call to main()
Lnew:	mov	lr, x1		    // simulate return address into _start in libdyld.dylib
	ldr     x0, [x28, #8] 	    // main param1 = argc
	add     x1, x28, #16	    // main param2 = argv
	add	x2, x1, x0, lsl #3  
	add	x2, x2, #8	    // main param3 = &env[0]
	mov	x3, x2
Lapple:	ldr	x4, [x3]
	add	x3, x3, #8
	cmp	x4, #0
	b.ne	Lapple		    // main param4 = apple
	br	x16

#endif // __arm64__
```

这里主要关注一下 `bl` 指令：

```asm6502
// call dyldbootstrap::start(app_mh, argc, argv, slide, dyld_mh, &startGlue)
bl	__ZN13dyldbootstrap5startEPK12macho_headeriPPKclS2_Pm
```

从注释了解到，其实就是调用 `dyldbootstrap::start()` 函数。在 dyldInitialization.cpp 中可以找到 `start` 函数的实现：

```c
//
//  This is code to bootstrap dyld.  This work in normally done for a program by dyld and crt.
//  In dyld we have to do this manually.
//
uintptr_t start(const struct macho_header* appsMachHeader, int argc, const char* argv[], 
				intptr_t slide, const struct macho_header* dyldsMachHeader,
				uintptr_t* startGlue)
{
	// if kernel had to slide dyld, we need to fix up load sensitive locations
	// we have to do this before using any global variables
	if ( slide != 0 ) {
		rebaseDyld(dyldsMachHeader, slide);
	}

	// allow dyld to use mach messaging
	mach_init();

	// kernel sets up env pointer to be just past end of agv array
	const char** envp = &argv[argc+1];
	
	// kernel sets up apple pointer to be just past end of envp array
	const char** apple = envp;
	while(*apple != NULL) { ++apple; }
	++apple;

	// set up random value for stack canary
	__guard_setup(apple);

#if DYLD_INITIALIZER_SUPPORT
	// run all C++ initializers inside dyld
	runDyldInitializers(dyldsMachHeader, slide, argc, argv, envp, apple);
#endif

	// now that we are done bootstrapping dyld, call dyld's main
	uintptr_t appsSlide = slideOfMainExecutable(appsMachHeader);
	return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}

```

`start` 函数中做了很多 dyld 初始化相关的工作，包括：

* rebaseDyld() dyld 重定位
* mach_init() mach消息初始化
* __guard_setup() 栈溢出保护

初始化工作完成后，此函数调用到了 `dyld::_main`，再将返回值传递给 `__dyld_start` 去调用真正的 `main()` 函数。

我们可以在 dyld.cpp 中找到` _main` 的实现， 代码比较长，就不贴代码了，不过我们可以看看 `_main` 函数的注释:

> Entry point for dyld.  The kernel loads dyld and jumps to __dyld_start which sets up some registers and call this function.
> 
> Returns address of main() in target program which __dyld_start jumps to

这个是说，内核加载 dyld，并跳转到 __dyld_start 函数，它主要设置一些寄存器，并且调用了 `_main`函数。这里刚好跟上面分析的过程相吻合。

`dyld::_mina()` 是应用程序启动的关机函数，主要做了以下一些事情：

1. 设置运行环境
2. 实例化主程序 
3. 加载共享缓存
4. 加载插入的动态库
5. 链接主程序
6. 链接插入的动态库
7. 执行弱符合绑定
8. 执行初始化方法
9. 查找入口并返回

##### 设置运行环境

这一步主要是设置运行参数、环境变量等。代码在开始的时候，将入参`mainExecutableMH` 赋值给了`sMainExecutableMachHeader`，这是一个`macho_header` 结构体，表示的是当前主程序的 Mach-O 头部信息，加载器依据 Mach-O 头部信息就可以解析整个 Mach-O 文件信息。接着调用 `setContext()` 设置上下文信息，包括一些回调函数、参数、标志信息等。如 `loadLibrary()` 函数实际调用的是 `libraryLocator()`，负责加载动态库。代码片断如下

```c
static void setContext(const macho_header* mainExecutableMH, int argc, const char* argv[], const char* envp[], const char* apple[])
{
    gLinkContext.loadLibrary			= &libraryLocator;
    gLinkContext.terminationRecorder	= &terminationRecorder;
    ...
}
```

##### 实例化主程序
```c
// instantiate ImageLoader for main executable
sMainExecutable = instantiateFromLoadedImage(mainExecutableMH, mainExecutableSlide, sExecPath);
```

这一步将主程序的 Mach-O 加载进内存，并实例化一个 ImageLoader。`instantiateFromLoadedImage()` 首先调用 `isCompatibleMachO()` 检测Mach-O 头部的magic、cputype、cpusubtype 等相关属性，判断 Mach-O 文件的兼容性，如果兼容性满足，则调用`ImageLoaderMachO::instantiateMainExecutable()` 实例化主程序的ImageLoader，代码如下：

```c
// The kernel maps in main executable before dyld gets control.  We need to 
// make an ImageLoader* for the already mapped in main executable.
static ImageLoader* instantiateFromLoadedImage(const macho_header* mh, uintptr_t slide, const char* path)
{
	// try mach-o loader
	if ( isCompatibleMachO((const uint8_t*)mh, path) ) {
		ImageLoader* image = ImageLoaderMachO::instantiateMainExecutable(mh, slide, path, gLinkContext);
		addImage(image);
		return image;
	}
	
	throw "main executable not a known format";
}
```

`ImageLoaderMachO::instantiateMainExecutable()` 函数里面首先会调用`sniffLoadCommands()` 函数来获取一些数据，包括：

* **compressed**：若Mach-O存在LC_DYLD_INFO和LC_DYLD_INFO_ONLY加载命令，则说明是压缩类型的Mach-O
* **segCount**：根据 LC_SEGMENT_COMMAND 加载命令来统计段数量。
* **libCount**：根据 LC_LOAD_DYLIB、LC_LOAD_WEAK_DYLIB、LC_REEXPORT_DYLIB、LC_LOAD_UPWARD_DYLIB 这几个加载命令来统计库的数量，库的数量不能超过4095个。
* **codeSigCmd**：通过解析LC_CODE_SIGNATURE来获取代码签名加载命令。
* **encryptCmd**：通过LC_ENCRYPTION_INFO和LC_ENCRYPTION_INFO_64来获取段的加密信息。

ImageLoader 是抽象类，其子类负责把 Mach-O 文件实例化为 image，当`sniffLoadCommands()` 解析完以后，根据 `compressed` 的值来决定调用哪个子类进行实例化，代码如下：

```c
if ( compressed ) 
    return ImageLoaderMachOCompressed::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
	else
#if SUPPORT_CLASSIC_MACHO
    return ImageLoaderMachOClassic::instantiateMainExecutable(mh, slide, path, segCount, libCount, context);
#else
```

`instantiateMainExecutable()` 执行完后，会调用 `addImage()` 函数将 image 加入到 `sAllImages` 全局镜像列表中。并将image映射到申请的内存中， 其代码如下：

```c
static void addImage(ImageLoader* image)
{
	// add to master list
    allImagesLock();
        sAllImages.push_back(image);
    allImagesUnlock();
	
	// update mapped ranges
	uintptr_t lastSegStart = 0;
	uintptr_t lastSegEnd = 0;
	for(unsigned int i=0, e=image->segmentCount(); i < e; ++i) {
		if ( image->segUnaccessible(i) ) 
			continue;
		uintptr_t start = image->segActualLoadAddress(i);
		uintptr_t end = image->segActualEndAddress(i);
		if ( start == lastSegEnd ) {
			// two segments are contiguous, just record combined segments
			lastSegEnd = end;
		}
		else {
			// non-contiguous segments, record last (if any)
			if ( lastSegEnd != 0 )
				addMappedRange(image, lastSegStart, lastSegEnd);
			lastSegStart = start;
			lastSegEnd = end;
		}		
	}
	if ( lastSegEnd != 0 )
		addMappedRange(image, lastSegStart, lastSegEnd);

	
	if ( sEnv.DYLD_PRINT_LIBRARIES || (sEnv.DYLD_PRINT_LIBRARIES_POST_LAUNCH && (sMainExecutable!=NULL) && sMainExecutable->isLinked()) ) {
		dyld::log("dyld: loaded: %s\n", image->getPath());
	}
	
}
```

##### 加载共享缓存

```c
// load shared cache
checkSharedRegionDisable();
#if DYLD_SHARED_CACHE_SUPPORT
if ( gLinkContext.sharedRegionMode != ImageLoader::kDontUseSharedRegion )
	mapSharedCache();
#endif
```

这一步先调用 `checkSharedRegionDisable()` 检查共享缓存是否禁用。该函数的iOS实现部分仅有一句注释，从注释我们可以推断iOS必须开启共享缓存才能正常工作。接下来调用 `mapSharedCache()` 来加载共享缓存。

##### 加载插入的动态库

```c
// load any inserted libraries
if ( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
    for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib) 
        loadInsertedDylib(*lib);
}
```

这一步是加载环境变量DYLD_INSERT_LIBRARIES中配置的动态库，先判断环境变量DYLD_INSERT_LIBRARIES中是否存在要加载的动态库，如果存在则调用 `loadInsertedDylib()` 依次加载。

`loadInsertedDylib()` 内部设置了一个LoadContext 后，调用了 `load()` 函数。该函数内部调用的一系列的 loadPhase*。

```c
// try all path permutations and check against existing loaded images
ImageLoader* image = loadPhase0(path, orgPath, context, NULL);
```

大致会按照下图的顺序搜索动态库，并调用不同的函数来继续处理。

![](02.png)

当内部调用到 `loadPhase5load()` 函数的时候，会先在共享缓存中搜索，如果存在则调用 `ImageLoaderMachO::instantiateFromCache()`  来实例化ImageLoader，否则通过 `loadPhase5open()` 打开文件并读取数据到内存后，再调用 `loadPhase6()` ，通过 `ImageLoaderMachO::instantiateFromFile()`  来实例化 ImageLoader，最后调用 `checkandAddImage()` 验证镜像并将其加入到全局镜像列表中。


##### 链接主程序

```c
// link main executable
gLinkContext.linkingMainExecutable = true;
link(sMainExecutable, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
```

这一步调用 `link()` 函数将实例化后的主程序进行动态修正，让二进制变为可正常执行的状态。`link()` 函数内部调用了`ImageLoader::link()` 函数，从源代码可以看到，这一步主要做了以下几个事情：

* `recursiveLoadLibraries()` 加载所有依赖的库到内存。
* `recursiveUpdateDepth()` 递归刷新依赖库的层级。
* `recursiveRebase()` 由于ASLR的存在，必须递归对主程序以及依赖库进行重定位操作。
* `recursiveBind()`  把主程序二进制和依赖进来的动态库全部执行符号表绑定。
* `weakBind()` 如果链接的不是主程序二进制的话，会在此时执行弱符号绑定，主程序二进制则在link()完后再执行弱符号绑定。
* `context.registerDOFs(dofs)` 注册DOF（DTrace Object Format）。

##### 链接插入的动态库

```c
// link any inserted libraries
// do this after linking main executable so that any dylibs pulled in by inserted 
// dylibs (e.g. libSystem) will not be in front of dylibs the program uses
if ( sInsertedDylibCount > 0 ) {
    for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
        ImageLoader* image = sAllImages[i+1];
        link(image, sEnv.DYLD_BIND_AT_LAUNCH, true, ImageLoader::RPathChain(NULL, NULL));
        image->setNeverUnloadRecursive();
    }
    // only INSERTED libraries can interpose
    // register interposing info after all inserted libraries are bound so chaining works
    for(unsigned int i=0; i < sInsertedDylibCount; ++i) {
        ImageLoader* image = sAllImages[i+1];
        image->registerInterposing();
    }
}
```

这一步与链接主程序一样，将前面调用 `addImage()` 函数保存在 `sAllImages` 中的动态库列表循环取出并调用 `link()` 进行链接，需要注意的是，`sAllImages` 中保存的第一项是主程序的镜像，所以要从 i+1的位置开始，取到的才是动态库的 ImageLoader。

接下来循环调用每个镜像的 `registerInterposing()` 函数，该函数会遍历Mach-O 的 LC_SEGMENT_COMMAND 加载命令，读取__DATA, __interpose，并将读取到的信息保存到 `fgInterposingTuples` 中。

##### 执行弱符合绑定

```c
// <rdar://problem/12186933> do weak binding only after all inserted images linked
sMainExecutable->weakBind(gLinkContext);
```

`weakBind()` 首先通过 `getCoalescedImages()` 合并所有动态库的弱符号到一个列表里，然后调用 `initializeCoalIterator()` 对需要绑定的弱符号进行排序，接着调用 `incrementCoalIterator()` 读取dyld_info_command 结构的 `weak_bind_off` 和 `weak_bind_size` 字段，确定弱符号的数据偏移与大小，最终进行弱符号绑定。

##### 执行初始化方法

```c
#if SUPPORT_OLD_CRT_INITIALIZATION
    // Old way is to run initializers via a callback from crt1.o
    if ( ! gRunInitializersOldWay ) 
        initializeMainExecutable(); 
#else
    // run all initializers
    initializeMainExecutable(); 
#endif
```

这一步由 `initializeMainExecutable()` 完成。dyld 会优先初始化动态库，然后初始化主程序。该函数首先执行 `runInitializers()`，内部再依次调用 `processInitializers()`、`recursiveInitialization()`。在 `processInitializers()` 之后会发送 `dyld_image_state_initialized` 通知。

在 `recursiveInitialization()` 的实现中有这么一行代码：

```c
// let objc know we are about to initialize this image
context.notifySingle(dyld_image_state_dependents_initialized, this);
```

注释告诉我们，这个函数主要目的是让 objc 知道 image 即将被初始化。之后执行初始化操作：

```c
// initialize this image
this->doInitialization(context);
```

在 `doInitialization()` 中首先调用了 `doImageInit()` ，然后调用 `doModInitFunctions()` 。

`doImageInit` 执行镜像的初始化函数，也就是 LC_ROUTINES_COMMAND中记录的函数，然后再执行 `doModInitFunctions` 来解析并执行_DATA_ 中__mod_init_func 这个 section 中保存的函数。_mod_init_funcs 中保存的是全局C++对象的构造函数以及所有带 `__attribute__((constructor)` 的C函数。

可以简单的写几行代码验证一下：

```
__attribute__((constructor))
void init_test() {
    printf("init_test called");
}

static TestClass t;
```

代码编译后可以使用 [MachOView](https://sourceforge.net/projects/machoview/)来查看 Mach-O 中的内容。

![](03.png)

可以看到 _mod_init_funcs 这个 section 中刚好有两个数据。

继续回到 `doInitialization()` 函数，在其实现中我们可以找到最终调用的方法：

```c
Initializer func = (Initializer)(((struct macho_routines_command*)cmd)->init_address + fSlide);
if ( context.verboseInit )
	dyld::log("dyld: calling -init function 0x%p in %s\n", func, this->getPath());
func(context.argc, context.argv, context.envp, context.apple, &context.programVars);
```

`Initializer`  是一个指向初始化方法的函数指针，这里的初始化方法就是上面 __mod_init_func 这个 section 中保存的函数。

我们可以通过添加 DYLD_PRINT_INITIALIZERS 环境变量在打印程序中依赖库的 `initializer` 方法：

![](04.png)

从打印中可以看到最先调用 libSystem.B.dylib 的 initializer :

```
**dyld: calling initializer function 0x7fff780ee94c in /usr/lib/libSystem.B.dylib**
```

可以从[这里](https://opensource.apple.com/source/Libsystem/Libsystem-169.3/init.c.auto.html)找到 libSystem 的 initializer 的完整实现。这里截取了部分代码：

```c
_libkernel_init(libkernel_funcs);

bootstrap_init();
mach_init();
pthread_init();
__libc_init(vars, libSystem_atfork_prepare, libSystem_atfork_parent, libSystem_atfork_child, apple);
__keymgr_initializer();
_dyld_initializer();
libdispatch_init();
```

这里我们只要关注一下 `libdispatch_init()` 函数。因为 `libdispatch_init()`  函数最终调用了 runtime 的初始化方法 `_objc_init`。我们可以打个符号断点来验证一下：

![](05.png)

这里可以看到 `_objc_init` 调用的顺序，先 `libSystem_initializer` 调用 `libdispatch_init` 再到 `_objc_init` 初始化 runtime。

这样从[这里](https://opensource.apple.com/source/libdispatch/libdispatch-913.60.2/)找到 `libdispatch_init()` 函数的实现。其中有这么几个函数调用： 

```c
_dispatch_hw_config_init();
_dispatch_time_init();
_dispatch_vtable_init();
_os_object_init();
_voucher_init();
_dispatch_introspection_init();
```

我们可以在 `_os_object_init()` 函数实现中发现确实调用了 `_objc_init()`：

```c
void
_os_object_init(void)
{
    _objc_init();
    ...
}
```

这个时候 runtime 被初始化了。

##### 查找入口点并返回

这一步调用主程序的 `getEntryFromLC_MAIN()`，从加载命令读取 LC_MAIN入口，如果没有 LC_MAIN 就调用 `getEntryFromLC_UNIXTHREAD()` 读取LC_UNIXTHREAD，找到后就跳到入口点指定的地址并返回。
至此，整个dyld的加载过程就完成了。


