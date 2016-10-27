title: "APK反逆向之二：四种基础加固方式"
date: 2016-10-15 11:28:37
categories: android
tags: [reserve, ndk, proguard, sign, dex, DexClassLoader]
toc: true
description: 近些年来移动 APP 数量呈现爆炸式的增长，黑产也从原来的PC端移到了移动端，伴随而来的逆向攻击手段也越来越高明。本篇章主要介绍应用加固的最基础的四种方式：1.proguard 混淆 2.签名比对验证 3.ndk 编译 .so 动态库 4.代码动态加载

---

##0x00 简介

应该大多数开发者都不会关注应用会不逆向破解，而且现在有第三方厂商提供免费的加固方案，所以 apk 应用的安全性就全部依赖于第三方。但是如果第三方加固方案被破解那么 apk 就陷于被动，所以我们也可以通过一些手段来加固应用本身逻辑，或者数据的加密。

最常见的加固就是 proguard 混淆，通过混淆可以加大逻辑分析的难度。对于数据的加密通过 ndk 开发动态库基本上可以防止大部分的破解者，逆向 native 跟逆向 java 不是一个级别的。

下面介绍的四种加固方式成本比较低，开发者可以很快开发完成，并且也不会影响应用的兼容性。

##0x01 proguard混淆

### 简介
ProGuard 是一个 SourceForge 上知名的开源项目。官网网址是：[http://proguard.sourceforge.net/](http://proguard.sourceforge.net/)。

java 编译后的字节码很容易被反编译，基本上 java 编译器都可以比如 eclipse、idea，还有用的比较多的 JD-GUI。都可以将 java 字节码转化为 java 源码。ProGuard的主要作用就是混淆。当然它还能对字节码进行缩减体积、优化等，但那些对于我们来说都算是次要的功能。

>ProGuard is a free Java class file shrinker, optimizer, obfuscator, and preverifier. It detects and removes unused classes, fields, methods, and attributes. It optimizes bytecode and removes unused instructions. It renames the remaining classes, fields, and methods using short meaningless names. The resulting applications and libraries are smaller, faster, and a bit better hardened against reverse engineering.

##0x02 签名比对验证

##0x03 ndk 编译.so 动态库

##0x04 代码动态加载