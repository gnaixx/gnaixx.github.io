title: "APK自我保护 - 代码乱序"
date: 2016-04-12 11:40:29
categories: android
tags: [android, crack]
toc: true
description: 代码乱序是指在不影响原有的代码逻辑，打乱代码布局，增加逆向分析的难度。代码乱序的原理与操作并不复杂，但是必须分析编译后的源码逻辑，而且实际运用到生成中难度比较大，所以这里也只是提供一个思路。

---

###乱序原理
为了增加逆向分析的难度,可以将原有代码在 smali 格式上进行乱序处理同时又不会影响程序的正常运行。乱序的基本原理如下图所示,将指令重新布局,并给每块指令赋予一个 label,在函数开头处使用 goto 跳到原先的第一条指令处,然后第一条指令处理完,再跳到 第二条指令,以此类推。

<img width=700px height=250px src="http://gnaix92.github.io/blog_images/apksafe/1.png" style="display:inline-block"/>

###乱序流程
下面步骤需要使用到的 `smali`, `baksmali` 和 `dex2jar` 工具可以在：[/tools](https://github.com/gnaix92/crack/tree/master/tools) 下载。
####Java代码
写了一个简单的Java代码：

```java
public class Hello {
	public static void main(String[] argc) {
		String a = "1";
		String b = "2";
		String c = a + b;
		System.out.println(c);
	}
}
```

####反编译.smali文件
我们最后是通过修改 smali 指令来达到重新布局代码，所以需要通过编译生成 smali 文件。

```java
javac Hello.java
dx --dex --output=Hello.dex Hello.class
java -jar baksmali-2.1.1.jar Hello.dex
```

1. 编译生成 Hello.class
2. dx 工具生成 Hello.dex
3. baksmali 工具生成 Hello.smali

经过上面命令会在跟目录下生成 `out/Hello.smali` 文件。


####修改smali文件
生成的 Hello.smali 代码为：

```smali
.class public LHello;
.super Ljava/lang/Object;
.source "Hello.java"


# direct methods
.method public constructor <init>()V
    .registers 1

    .prologue
    .line 1
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V

    return-void
.end method

.method public static main([Ljava/lang/String;)V
    .registers 4

    .prologue
    .line 3
    const-string v0, "1"

    .line 4
    const-string v1, "2"

    .line 5
    new-instance v2, Ljava/lang/StringBuilder;

    invoke-direct {v2}, Ljava/lang/StringBuilder;-><init>()V

    invoke-virtual {v2, v0}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    move-result-object v0

    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    move-result-object v0

    invoke-virtual {v0}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v0

    .line 6
    sget-object v1, Ljava/lang/System;->out:Ljava/io/PrintStream;

    invoke-virtual {v1, v0}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    .line 7
    return-void
.end method
```
其中我们需要关注的是 main 函数的内部流程，main 函数主要是分了三个步骤： 
   
- 定义字符串常量
- 拼接字符
- 打印字符串

<img width=700px height=500px src="http://gnaix92.github.io/blog_images/apksafe/2.png" style="display:inline-block"/>


修改 smali 代码顺序（只要修改 main 函数其他不变）：

```smali
.method public static main([Ljava/lang/String;)V
    .registers 4

    .prologue

    goto :lab1

    :lab3

    sget-object v1, Ljava/lang/System;->out:Ljava/io/PrintStream;

    invoke-virtual {v1, v0}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    goto :end

    :lab2

    new-instance v2, Ljava/lang/StringBuilder;

    invoke-direct {v2}, Ljava/lang/StringBuilder;-><init>()V

    invoke-virtual {v2, v0}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    move-result-object v0

    invoke-virtual {v0, v1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    move-result-object v0

    invoke-virtual {v0}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v0

    goto :lab3

    :lab1
    
    const-string v0, "1"

    const-string v1, "2"

    goto :lab2

    :end

    return-void
.end method
```

修改后的代码顺序：
<img width=700px height=500px src="http://gnaix92.github.io/blog_images/apksafe/3.png" style="display:inline-block"/>


####重新编译回dex文件
重新编译回 dex 文件很简单，只需要 smali 工具就可以了：

```java
java -jar smali-2.1.1.jar Hello.smali
```

运行完成会生成一个 `out.dex` 文件。


####代码对比
生成的 `out.dex` 不能直接用 `jd-gui` 查看源码，所以可以 `dex2jar` 工具转化为 jar 文件:

```java
d2j-dex2jar.sh out.dex
```

<img src="http://gnaix92.github.io/blog_images/apksafe/4.png" style="height:200px; width:700px"/>

<img src="http://gnaix92.github.io/blog_images/apksafe/5.png" style="height:200px; width:700px;"/>


上边为未做处理的反编译代码，下边为乱序后的代码。可以看出右边的代码逻辑相对来说会比较难懂。