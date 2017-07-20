---
title: DEX文件加载流程分析
tags: [android, dex, classload]
toc: true
date: 2017-02-22 14:40:58
categories: android
description: 系统 API 不管 DexClassLoader 还是 PathClassLoader 都只支持文件路径参数，所以在加载 dex 文件的时候必须生成一个缓存文件，而且是正常格式的 dex 文件。自定义的目的就是去掉这个缓存的过程，简单来讲就是让 DexClassLoader 支持 byte 数组参数，以内存形式加载。目前只实现了 Android4.0 系统到 Android4.4 系统真正意义上的自定义 Load。

---
## 0x00 前言
通过搜索和阅读不同版本的源码，发现不同版本对Dex加载在Java层都大同小异，区别比较明显的是cookie(Dex加载后返回的一个值)的类型一直在变化，从int到long再到Object。

区别比较大的是native层对Dex的解析处理，每个大版本处理的方式都有差异。其中4.0版本提供了一个接口可以以byte数组的方式加载Dex，但是art模式这个方法就取消了，后续版本也取消了。

本文也只通过4.0的接口实现了自定义Dex，其他版本的加载方式通过NDK反射调用实现。

**源码地址:**[https://github.com/gnaixx/hidex-hack](https://github.com/gnaixx/hidex-hack/tree/master/hidex-load)

## 0x01 Load流程
重写Dex的加载需要弄清楚Load的流程(以下代码以6.0为例)。在调用DexClassLoader进行dex文件的Load，主要用到了下面的两个函数：

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

### Dex文件加载
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
 
### Class类加载
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

### 流程图
上述的流程就是 dex 文件在 java 层加载的过程，可以用一个流程图直观的展示函数调用流程。
![dex-load](/blog_images/20170222/dex-load.png)

## 0x02 源码分析
通过上面流程的梳理可以看出我们需要关注DexFile文件中的两个函数: **openDexFileNative** 和 **defineClassNative** 不同版本这个两个函数的实现也是不一样的，主要是**openDexFileNative**。

### Android 2.3 以下系统

```java
/**
 * Loads a class. Returns the class on success, or a {@code null} reference
 * on failure.
 * <p>
 * If you are not calling this from a class loader, this is most likely not
 * going to do what you want. Use {@link Class#forName(String)} instead.
 * <p>
 * The method does not throw {@link ClassNotFoundException} if the class
 * isn't found because it isn't feasible to throw exceptions wildly every
 * time a class is not found in the first DEX file we look at. It will
 * throw exceptions for other failures, though.
 *
 * @param name
 *            the class name, which should look like "java/lang/String"
 *
 * @param loader
 *            the class loader that tries to load the class (in most cases
 *            the caller of the method
 *
 * @return the {@link Class} object representing the class, or {@code null}
 *         if the class cannot be loaded
 *
 * @cts Exception comment is a bit cryptic. What exception will be thrown?
 */
public Class loadClass(String name, ClassLoader loader) {
    String slashName = name.replace('.', '/');
    return loadClassBinaryName(slashName, loader);
}

/**
 * See {@link #loadClass(String, ClassLoader)}.
 *
 * This takes a "binary" class name to better match ClassLoader semantics.
 *
 * {@hide}
 */
public Class loadClassBinaryName(String name, ClassLoader loader) {
    return defineClass(name, loader, mCookie,
        null);
        //new ProtectionDomain(name) /*DEBUG ONLY*/);
}

native private static Class defineClass(String name, ClassLoader loader,
    int cookie, ProtectionDomain pd);

/*
 * Open a DEX file.  The value returned is a magic VM cookie.  On
 * failure, an IOException is thrown.
 */
native private static int openDexFile(String sourceName, String outputName,
    int flags) throws IOException;
```

### 14<API<18

### API=19

### 5.0-5.1(api:21-22)系统

### 6.0(api:23)系统

### 7.0+(api:24)系统

## 0x03 代码实现

## 0x04 总结
