title: "NDK开发 - JNI数据类型与Java数据类型映射关系"
date: 2016-03-30 17:18:34
categories: android
tags: [android, ndk, jni]
toc: true
description: 介绍完Android JNI的开发流程就要开始码代码了，不过在码代码前还是有必要了解下JNI数据类型与Java数据类型映射关系，直接开始写肯定会懵逼。

---
　　在调用 Java native 方法将实参传递给 C/C++ 函数的时候，会自动将 java 形参的数据类型自动转换成 C/C++ 相应的数据类型，所以我们在写 JNI 程序的时候，必须要明白它们之间数据类型的对应关系。

　　在 Java 语言中数据类型分为基本数据类型和引用类型，同样JNI中也对应着基础数据类型和引用类型。

###基本数据类型
　　Java 中基本数据类型包括：byte, char, short, int, long, float, double, boolean。对应JNI数据类型的：jbyte, jchar, jshort, jint, jfloat, jdoubule, jboolean。    
下面是JNI规范文档中描述 Java 与 JNI数据类型的对应关系：

|  java language type  |   native   |  description  |
| -------------------- |   -------- |  ------------ |
|boolean|jboolean|unsigned 8 bits|
|byte|jbyte|signed 8 bits|
|char|jchar|unsigned 16 bits|
|short|jshort|signed 16 bits|
|int|jint|signed 32 bits|
|long|jlong|signed 64 bits|
|float|jfloat|32 bits|
|double|jdouble|64 bits|

###引用类型
　　Java语言中除了上述的8中基本数据类型外其他都是引用类型：Object，String, 数组等。    
　　所有的JNI引用类型全部是jobject类型，为了使用方便和类型安全，JNI 定义了一个引用类型集合，集合当中的所有类型都是 jobject 的子类，这些子类和 Java 中常用的引用类型相对应。例如：jstring 表示字符串、jclass 表示 class 字节码对象、jthrowable 表示异常、jarray 表示数组，另外 jarray 派生了 8 个子类，分别对应Java 中的 8 种基本数据类型（jintArray、jshortArray、jlongArray等）。

引用类型对应关系：    
![http://gnaix92.github.io/blog_images/ndk/3.png](http://gnaix92.github.io/blog_images/ndk/3.png)