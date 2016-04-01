title: "AndroidManifest中sdkVersion含义"
date: 2015-11-08 23:16:22
categories: android
tags: [android, manifest]
toc: true
description: 在android开发中你需要配置不同的Sdk版本，每个版本都有其特定的含义
---

在android开发中你需要配置不同的Sdk版本，每个版本都有其特定的含义。

###Eclipse
在使用eclipse开发android应用程序时可以发现AndroidManifest.xml文件中有下面几行代码

```java
<uses-sdk
        android:minSdkVersion="14"
        android:targetSdkVersion="23" />
```
另外在project.propert文件中定义了compileSdkVersion

```java
target=android-23
```

###Android Studio/IDEA
而在使用Android studio/idea开发时不在使用Ant作为自动化构建工具改用gradle，所以大部分配置信息都在build.gradle文件中。

```java
 compileSdkVersion 23
 buildToolsVersion "23.0.2"

 defaultConfig {
    applicationId "com.example.gnaix.material"
    minSdkVersion 14
    targetSdkVersion 23
    versionCode 1
    versionName "1.0"
 }
```
虽然我很喜欢as，但是必须说一句as的编译速度比eclipse慢好多。

###***SdkVersion含义

1. compileSdkVersion: 编译源码的Android SDK版本号
2. minSdkVersion: apk最低支持版本
3. maxSdkVersion: apk最高支持版本(一般不设置)
4. targetSdkVersion: 指点了具体的api，在该版本设备上运行时只会调用该版本实现的API，而不是早期版本的API（上一篇介绍permission中有提到该配置）   
5. buildTooVersion: 构建工具的版本号，其中包括了打包工具aapt、dx等。工具目录位于sdk_path/build-tools/xx.xx.xx。可以用高版本build-tool去构建一个低版本的sdk工程