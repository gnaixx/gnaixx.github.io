---
title: 自定义 DexClassLoad(伪)
tags: [android, dex, classload]
toc: true
date: 2017-02-22 14:40:58
categories: android
description: 系统 API 不管 DexClassLoader 还是 PathClassLoader 都只支持文件路径参数，所以在加载 dex 文件的时候必须生成一个缓存文件，而且是正常格式的 dex 文件。自定义的目的就是去掉这个缓存的过程，简单来讲就是让 DexClassLoader 支持 byte 数组参数，以内存形式加载。为什么加个伪，因为只实现了 Android4.0 系统到 Android4.4 系统，对于其他版本的系统算不上真正意义上的自定义。

---
## 0x00 前言

## 0x01 Load流程
想要重写DexClassLoader先要弄清楚它的Load流程(以下代码以6.0为例)。用DexClassLoader进行dex文件的Load，主要用到了两个函数：

```java
/
 * DexClassLoader 构造函数
 *
 * @param dexPath            加载的 dex 文件路径
 * @param optimizedDirectory 优化后 dex 文件路径
 * @param libraryPath        依赖的 native 库路径
 * @param parent             父 ClassLoader
 */
public DexClassLoader(String dexPath， String optimizedDirectory， String libraryPath， ClassLoader parent){}

/
 * 根据类名 Loads Class 
 *
 * @param className 类名
 */
public Class<?> loadClass(String className) throws ClassNotFoundException {}

//
//示例代码:
DexClassLoader classLoader = new DexClassLoader(
        dexFile.getPath()，
        tempPath，
        null，
        context.getClassLoader());

Class clazz = classLoader.loadClass(className);
```

上面操作可以分为两部分：1.dex文件加载 2.class类加载。可以分析这个两个步骤的流程。

### dex文件加载
从DexClassLoader.java开始，该类继承于BaseClassLoader，内部只实现了一个构造函数，并且直接调用父类的构造函数(以下源码去掉了部分注释)。     
DexClassLoader.java:
![DexClassLoader](/blog_images/20170222/dexclassloader.png)

BaseDexClassLoader构造函数创建了一个DexPathList对象，主要是负责解析加载的 DEX 文件的解析。      
BaseDexClassLoader.java:
![BaseDexClassLoader](/blog_images/20170222/basedexclassloader.png)

DexPathList构造函数中调用了makePathElements解析了 dex 文件。    
DexPathList.java:
![DexPathList](/blog_images/20170222/dexpathlist.png)

makeDexElements中调用loadDexFile加载文件，并返回一个DexFile对象。     
DexPathList.java:
![makeDexElements](/blog_images/20170222/makedexelements.png)

loadDexFile中如果不存在optimizedDirectory直接new一个DexFile，如果存在则调用DexFile.loadDex方法返回一个DexFile对象。    
DexPathList.java:
![loadDexFile](/blog_images/20170222/loaddexfile.png)

DexFile中有多个构造函数，上面DexPathList.loadDexFile调用了其中两个，可以看到最后关键的代码都是openDexFile函数。     
DexFile.java:
![DexFile](/blog_images/20170222/dexfile.png)

openDexFile是 Load dex文件的关键，他的实现很简单只调用了一个native方法openDexFileNative这个方法不同版本的 API 实现都有差别，下一章节会介绍到。    
DexFile.java:
![openDexFile](/blog_images/20170222/opendexfile.png)

到这里dex文件的加载java层的流程基本就结束了。
 
### class类加载
DexClassLoader加载一个类是通过loadClass方法，但是在DexClassLoader和BaseDexClassLoader中都没有该方法，该方法的实现是在ClassLoader中。loadClass中有三处返回，1.判断BootClassLoader是否加载 2.父类ClassLoader是否加载 3.通过findClass加载。    
ClassLoader.java:
![loadClass](/blog_images/20170222/loadclass.png)

ClassLoader.findClass实现是空的，真正的实现在BaseDexClassLoader中，调用了DexPathList的findClass方法。    
BaseDexClassLoader.java:
![bfindClass](/blog_images/20170222/bfindclass.png)

DexPathList通过遍历所有加载的DexFile对象，调用DexFile的loadClassBinaryName方法。    
DexPathList.java:
![dfindClass](/blog_images/20170222/dfindclass.png)

loadClassBinaryName方法直接调用了defineClass方法，而defineClass调用了一个 native 方法defineClassNative。其中第三个参数cookie即方法openDexFile的返回值。     
DexFile.java:
![loadClassBinaryName](/blog_images/20170222/loadclassbinaryname.png)

上述的流程就是 dex 文件在 java 层加载的过程，可以用一个流程图直观的展示函数调用流程。
![dex-load](/blog_images/20170222/dex-load.png)

## 0x02 源码分析

### 2.3(api:10)以下系统

### 4.0-4.3(api:14-18)系统

### 4.4(api:19)系统

### 5.0-5.1(api:21-22)系统

### 6.0(api:23)系统

### 7.0+(api:24)系统

## 0x03 代码实现

## 0x04 总结
