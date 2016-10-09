title: "APK自我保护 - 字符串处理"
date: 2016-04-22 11:06:58
categories: android
tags: [android, hardening]
toc: true
description: 字符串处理可以增加静态分析的难度，但是这种增加是有限的。只要破解者有足够的耐心同样可以破解。

---

在开发过程中字符串不可避免，但是这些字符串也可能是破解的关键点，比如服务器的地址和错误提示这些敏感的字符串信息。如果这些字符串采用硬编码方式，很容易通过静态分析获取。之前的一篇blog以提示的字符串以突破点 [Android程序逆向分析](https://gnaix92.github.io/2016/03/06/android-reverse-engineering/)

###0x00 普通方式定义字符串
**Java 中定义一个字符串：**

```java
private String normalString(){
    String str = "Hello world";
    return str;
}
```

**反编译的.smali代码**

```smali
.method private normalString()Ljava/lang/String;
    .registers 2

    .prologue
    .line 16
    const-string v0, "Hello world"

    .line 17
    .local v0, "str":Ljava/lang/String;
    return-object v0
.end method
```

可以看出来 `const-string` 关键字后面就是定义的字符串值，甚至可以使用自动化分析工具批量提取。

###0x01 解决方案
####1. StringBuilder 拼接
StringBuilder 类通过 append 方法来构造需要的字符串。这种方式可以增加自动化分析的难度，如果要获取完整的字符串就必须进行相应的词法语法解析了。
    
**Java 中拼接字符串代码：**

```java
private String buildString(){
    StringBuilder builder = new StringBuilder();
    builder.append("Hello");
    builder.append(" ");
    builder.append("world");
    return builder.toString();
}
```

**反编译的.smali代码:**

```smali
.method private buildString()Ljava/lang/String;
    .registers 3

    .prologue
    .line 21
    new-instance v0, Ljava/lang/StringBuilder;

    invoke-direct {v0}, Ljava/lang/StringBuilder;-><init>()V

    .line 22
    .local v0, "builder":Ljava/lang/StringBuilder;
    const-string v1, "Hello"

    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    .line 23
    const-string v1, " "

    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    .line 24
    const-string v1, "world"

    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    .line 25
    invoke-virtual {v0}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v1

    return-object v1
.end method
```

可以看出反编译后的 .smali 代码对破解增加了一定的难度，并不能一眼就识别出来。

####2. 编码混淆
编码混淆是在硬编码的时候将字符串先转换成 **16进制** 的数组或者 **Unicode** 编码，在使用的时候在转回字符串。这种方式在反编译成 .smali 代码比 StringBuilder 方式更难直接识别。

**Java代码:**   

```java
private String encodeString(){
    byte[] strBytes = {0x48, 0x65, 0x6c, 0x6c, 0x6f, 0x20, 0x77, 0x6f, 0x72, 0x6c, 0x64};
    String str = new String(strBytes);
    return str;
}
``` 

**反编译 .smali 代码:**

```smali
.method private encodeString()Ljava/lang/String;
    .registers 4

    .prologue
    .line 29
    const/16 v2, 0xb

    new-array v1, v2, [B

    fill-array-data v1, :array_e

    .line 30
    .local v1, "strBytes":[B
    new-instance v0, Ljava/lang/String;

    invoke-direct {v0, v1}, Ljava/lang/String;-><init>([B)V

    .line 31
    .local v0, "str":Ljava/lang/String;
    return-object v0

    .line 29
    nop

    :array_e
    .array-data 1
        0x48t
        0x65t
        0x6ct
        0x6ct
        0x6ft
        0x20t
        0x77t
        0x6ft
        0x72t
        0x6ct
        0x64t
    .end array-data
.end method
```
.smali 代码中可以看出已经隐藏了所有字符。

####3. 加密处理
加密处理是先将字符串在本地进行加密处理，后将密文硬编码进去，运行时载进行解密。    
**加密步骤:**

- 字符串加密
- 硬编码进程序
- 编译运行
- 解密密文

当然因为 Java 代码相对来说比较容易反编译，并且该方式需要将解密方法放在 APK 本地，所以我们可以将解密方法通过 JNI 实现，加大反编译难度。

###0x02总结
任何一种加固方式都只是加大了反破解的难度，并不能完全避免 Android 程序被破解。