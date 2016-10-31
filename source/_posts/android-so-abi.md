title: "关于Android so文件你所需要了解的"
date: 2016-09-07 18:27:02
categories: android
tags: [android, ndk, so, abi]
toc: true
description: 现在很多第三方SDK都会提供 so, 关于 ndk开发的好处之前的文章也有做了介绍。但是在技术支持的过程中发现大多数的Android 开发其实并不知道怎么正确的集成so文件。一般技术都是把第三方提供的所有so都导入到工程中，这样做对应单个第三方so是没有问题的，但是如果集成了多个第三方问题就来了。

---

##0x00 前言
早期的Android系统几乎只支持ARMv5的CPU架构。你知道现在它支持多少种吗？7种！

Android系统目前支持以下七种不同的CPU架构：ARMv5，ARMv7 (从2010年起)，x86 (从2011年起)，MIPS (从2012年起)，ARMv8，MIPS64和x86_64 (从2014年起)，每一种都关联着一个相应的ABI。

应用程序二进制接口（Application Binary Interface）定义了二进制文件（尤其是.so文件）如何运行在相应的系统平台上，从使用的指令集，内存对齐到可用的系统函数库。在Android系统上，每一个CPU架构对应一个ABI：armeabi，armeabi-v7a，x86，mips，arm64-v8a，mips64，x86_64。

##0x01 为什么你需要重点关注.so文件
如果项目中使用到了NDK，它将会生成.so文件，因此显然你已经在关注它了。如果只是使用Java语言进行编码，你可能在想不需要关注.so文件了，因为Java是跨平台的。但事实上，即使你在项目中只是使用Java语言，很多情况下，你可能并没有意识到项目中依赖的函数库或者引擎库里面已经嵌入了.so文件，并依赖于不同的ABI。

例如，项目中使用RenderScript支持库，OpenCV，Unity，android-gif-drawable，SQLCipher等，你都已经在生成的APK文件中包含.so文件了，而你需要关注.so文件。

Android应用支持的ABI取决于APK中位于lib/ABI目录中的.so文件，其中ABI可能是上面说过的七种ABI中的一种。    
![1.png](https://gnaixx.github.io/blog_images/so-abi/1.png) 

[Native Libs Monitor](https://play.google.com/store/apps/details?id=com.xh.nativelibsmonitor.app)这个应用可以帮助我们理解手机上安装的APK用到了哪些.so文件，以及.so文件来源于哪些函数库或者框架。

###ABI和CPU的关系

很多设备都支持多于一种的ABI。例如ARM64和x86设备也可以同时运行armeabi-v7a和armeabi的二进制包。但最好是针对特定平台提供相应平台的二进制包，这种情况下运行时就少了一个模拟层（例如x86设备上模拟arm的虚拟层），从而得到更好的性能（归功于最近的架构更新，例如硬件fpu，更多的寄存器，更好的向量化等）。

我们可以通过Build.SUPPORTED_ABIS得到根据偏好排序的设备支持的ABI列表。但你不应该从你的应用程序中读取它，因为Android包管理器安装APK时，会自动选择APK包中为对应系统ABI预编译好的.so文件，如果在对应的lib／ABI目录中存在.so文件的话。

CPU/ABI|armeabi|armeabi-v7a|arm64-v8a|x86|x86_64|mips|mips64|
-------|-------|-----------|---------|---|------|----|------|
ARMv5  |支持    |　         |　       |　  |　    |　   |　    |
ARMv7  |支持    |支持        |　       |　  |　    |　   |　    |
ARMv8  |支持    |支持        |支持      |　  |　    |　   |　    |
x86    |支持    |支持        |　        |支持|　    |　   |　    |
x86_64 |支持    |　       　 |　　　     |支持|支持 |　   |　    |
mips   |　     |　         |　        |　　|　    |支持 |　    |
mips_64|　     |　         |　        |　　|　    |支持 |支持   |


##0x02 App中可能出错的地方
处理.so文件时有一条简单却并不知名的重要法则。

你应该尽可能的提供专为每个ABI优化过的.so文件，但要么全部支持，要么都不支持：你不应该混合着使用。你应该为每个ABI目录提供对应的.so文件。

当一个应用安装在设备上，只有该设备支持的CPU架构对应的.so文件会被安装。在x86设备上，libs/x86目录中如果存在.so文件的话，会被安装，如果不存在，则会选择armeabi-v7a中的.so文件，如果也不存在，则选择armeabi目录中的.so文件（因为x86设备也支持armeabi-v7a和armeabi）。

##0x03 其他出错情况
当你引入一个.so文件时，不止影响到CPU架构。我从其他开发者那里可以看到一系列常见的错误，其中最多的是"UnsatisfiedLinkError"，"dlopen: failed"以及其他类型的crash或者低下的性能：

##0x04 使用android-21平台版本编译的.so文件运行在android-15的设备上
使用NDK时，你可能会倾向于使用最新的编译平台，但事实上这是错误的，因为NDK平台不是后向兼容的，而是前向兼容的。推荐使用app的minSdkVersion对应的编译平台。

这也意味着当你引入一个预编译好的.so文件时，你需要检查它被编译所用的平台版本。

##0x05 混合使用不同C++运行时编译的.so文件
.so文件可以依赖于不同的C++运行时，静态编译或者动态加载。混合使用不同版本的C++运行时可能导致很多奇怪的crash，是应该避免的。作为一个经验法则，当只有一个.so文件时，静态编译C++运行时是没问题的，否则当存在多个.so文件时，应该让所有的.so文件都动态链接相同的C++运行时。

这意味着当引入一个新的预编译.so文件，而且项目中还存在其他的.so文件时，我们需要首先确认新引入的.so文件使用的C++运行时是否和已经存在的.so文件一致。

##0x06 没有为每个支持的CPU架构提供对应的.so文件
这一点在前文已经说到了，但你应该真的特别注意它，因为它可能发生在根本没有意识到的情况下。

例如：你的app支持armeabi-v7a和x86架构，然后使用Android Studio新增了一个函数库依赖，这个函数库包含.so文件并支持全部的CPU架构，例如新增android-gif-drawable函数库：

```script
compile ‘pl.droidsonroids.gif:android-gif-drawable:1.1.+’
```

发布我们的app后，会发现它在某些设备上会发生Crash，例如Galaxy S6，最终可以发现只有64位目录下的.so文件被安装进手机。

解决方案：重新编译我们的.so文件使其支持缺失的ABIs，或者设置 `ndk.abiFilters`显示指定支持的ABIs。

最后一点：**如果你是一个SDK提供者，但提供的函数库不支持所有的ABIs，那你将会搞砸你的用户，因为他们能支持的ABIs必将只能少于你提供的。**

##0x07 将.so文件放在错误的地方
我们往往很容易对.so文件应该放在或者生成到哪里感到困惑，下面是一个总结：

- Android Studio工程放在jniLibs/ABI目录中（当然也可以通过在build.gradle文件中的设置jniLibs.srcDir属性自己指定）
- Eclipse工程放在libs/ABI目录中（这也是ndk-build命令默认生成.so文件的目录）
- AAR压缩包中位于jni/ABI目录中（.so文件会自动包含到引用AAR压缩包的APK中）
- 最终APK文件中的lib/ABI目录中
- 通过PackageManager安装后，在小于Android 5.0的系统中，.so文件位于app的nativeLibraryPath目录中；在大于等于Android 5.0的系统中，.so文件位于app的nativeLibraryRootDir/CPU_ARCH目录中。

##0x08 只提供armeabi架构的.so文件而忽略其他ABIs的
所有的x86/x86_64/armeabi-v7a/arm64-v8a设备都支持armeabi架构的.so文件，因此似乎移除其他ABIs的.so文件是一个减少APK大小的好技巧。但事实上并不是：这不只影响到函数库的性能和兼容性。

x86设备能够很好的运行ARM类型函数库，但并不保证100%不发生crash，特别是对旧设备。64位设备（arm64-v8a, x86_64, mips64）能够运行32位的函数库，但是以32位模式运行，在64位平台上运行32位版本的ART和Android组件，将丢失专为64位优化过的性能（ART，webview，media等等）。

以减少APK包大小为由是一个错误的借口，因为你也可以选择在应用市场上传指定ABI版本的APK，生成不同ABI版本的APK可以在build.gradle中如下配置：

```gradle
android {
    ... 
    splits {
        abi {
            enable true
            reset()
            include 'x86', 'x86_64', 'armeabi-v7a', 'arm64-v8a' //select ABIs to build APKs for
            universalApk true //generate an additional APK that contains all the ABIs
        }
    }

    // map for the version code
    project.ext.versionCodes = ['armeabi': 1, 'armeabi-v7a': 2, 'arm64-v8a': 3, 'mips': 5, 'mips64': 6, 'x86': 8, 'x86_64': 9]

    android.applicationVariants.all { variant ->
        // assign different version code for each output
        variant.outputs.each { output ->
            output.versionCodeOverride =
                    project.ext.versionCodes.get(output.getFilter(com.android.build.OutputFile.ABI), 0) * 1000000 + android.defaultConfig.versionCode
        }
    }
 }
```

##0x09 使用系统私有库
尽量使用NDK提供的函数，其他第三方库可能不能正常在设备上运行，也可能导致异常奔溃。



参考文献:[What you should know about .so files](http://ph0b.com/android-abis-and-so-files/)    
