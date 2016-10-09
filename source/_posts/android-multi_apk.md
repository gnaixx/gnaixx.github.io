title: "APK多开原理"
date: 2016-04-11 14:06:09
categories: android
tags: [android, crack]
toc: true
description: 去年给运营同事解释了最简单的多开原理，最简单的方式通过修改包名达到多开的目的。很多 APK 现在已经不能通过这个方式达到多开了，虽然 LOW ,但是对了解多开原理还是有点用的。

---
　　所有的 Android 应用程序都有一个包名。包名是设备上的这个应用程序的唯一标识，也是在谷歌Play商店上的唯一标识。所以多开也就是基于这个原理来实现。在这之前需要了解下`ApplicationId` 和 `PackageName` 的区别。这里有一份官方文档: [http://tools.android.com/tech-docs/new-build-system/applicationid-vs-packagename](http://tools.android.com/tech-docs/new-build-system/applicationid-vs-packagename)

###0x00 Application 和 PackageName

　　在旧版本的 Android Gradle 和 Eclipse Ant 构建系统中，应用程序的包名是由 AndroidManifest 文件的根元素的 package 属性决定的：   

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.my.app"
    android:versionCode="1"
    android:versionName="1.0" >
```
但其实这里的 package 有两种含义：

- 应用程序包名
- 资源类的包名（以及解析任何相关的Activity的类名）

　　Android 开发者应该都知道 R.java, 每个资源文件都会在 R 类中生成唯一的16进制编号。上面例子中将会生成 **com.example.my.app.R**, 如果需要在代码中引入资源，就需要导入 **com.example.my.app.R**。

　　在新版本的 Android Gradle 构建系统可以通过 flavors 构建不同版本的的应用程序；比如我们可以同时打包 “free” 版本和 “pro” 版本，并且这两个版本在不同的应用商店上有不同的包名，所以他们可以被同时安装在同一个机器上面。

　　因此新版本构建工具多了一个 `ApplicationId`。新增的 `ApplicationId` 表示应用程序的包名，而 `package` 用于在源代码中来引用您的R类的，并且是解析任何相关的Activity/Service 注册的包。    

在项目的 `app/build.gradle`中指定 `ApplicationId`:   

```java
apply plugin: 'com.android.application'

android {
    compileSdkVersion 19
    buildToolsVersion "19.1"

    defaultConfig {
        applicationId "com.example.my.app"
        minSdkVersion 15
        targetSdkVersion 19
        versionCode 1
        versionName "1.0"
    }
    ...
```

`package` 依旧在 AndroidManifest.xml 中定义。

　　packageName 在代码中使用，通常在 AndroidManifest.xml中指定，applicationId 则只是用于程序的标识，通常在 build.gradle 中指定。这样有一个好处，假如你想发布一个免费版，一个收费版，你只需要在 build.gradle 中把applicationId 后面加上免费版的后缀包名（如".free"），收费版加上收费版的后缀即可，而不需要修改你的其他代码。    
　　你可以通过使用以下的 Gradle DSL 方法，为不同的flavors和构建类型修改你的应用程序的 ApplicationId：

```java
 productFlavors {
        pro {
            applicationId = "com.example.my.pkg.pro"
        }
        free {
            applicationId = "com.example.my.pkg.free"
        }
    }

    buildTypes {
        debug {
            applicationIdSuffix ".debug"
        }
    }
    ....
```

<font color="red">**PS:**</font> 出于兼容性原因，如果您没有在您的 build.gradle 文件中定义 applicationId，这个applicationId 将默认为 AndroidManifest.xml 中所指定的相同的值。

###0x01 Android Studio 项目多开分析
####修改 app/build.gradle 文件中的 applicationId
<img width=700px height=300px src="https://gnaix92.github.io/blog_images/multi/1.png" style="display:inline-block"/>

####同时安装
<img width=700px height=200px src="https://gnaix92.github.io/blog_images/multi/2.png" style="display:inline-block"/>

**查看/data/data目录**    
可以看到这时候已经有两个安装包了。
<img width=700px height=200px src="https://gnaix92.github.io/blog_images/multi/3.png" style="display:inline-block"/>

**打开应用获取包名（“^^”后面为版本号）**
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/4.png" style="display:inline-block"/>
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/5.png" style="display:inline-block"/>

####利用 apktool 反编译查看 AndroidManifest.xml 文件 
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/6.png" style="display:inline-block"/>
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/7.png" style="display:inline-block"/>
可以看到编译生成的 package 属性已经变成了 applicationId 设定的值

####利用 dex2jar 反编译 classes.dex 代码
<img width=700px height=200px src="https://gnaix92.github.io/blog_images/multi/8.png" style="display:inline-block"/>
两个 APK 的 classse.dex 代码是一样的，没有变化。


###0x02 Eclipse 项目分析

####修改 AndroidManifest.xml 中的 package 属性
　　因为 Eclipse 构建项目是 package 属性是表示包名的同时也是资源文件的包名，所以改动后 R 文件包名也会变化。
<img width=400px height=300px src="https://gnaix92.github.io/blog_images/multi/9.png" style="display:inline-block"/>

<img width=700px height=300px src="https://gnaix92.github.io/blog_images/multi/10.png" style="display:inline-block"/>

<img width=700px height=300px src="https://gnaix92.github.io/blog_images/multi/11.png" style="display:inline-block"/>

####同时安装
<img width=700px height=300px src="https://gnaix92.github.io/blog_images/multi/12.png" style="display:inline-block"/>

**查看/data/data目录**    
可以看到这时候已经有两个安装包了。
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/13.png" style="display:inline-block"/>

**打开应用获取包名（“^^”后面为版本号）**
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/14.png" style="display:inline-block"/>
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/15.png" style="display:inline-block"/>

####利用 apktool 反编译查看 AndroidManifest.xml 文件 
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/16.png" style="display:inline-block"/>
<img width=700px height=150px src="https://gnaix92.github.io/blog_images/multi/17.png" style="display:inline-block"/>


####利用 dex2jar 反编译 classes.dex 代码
<img width=700px height=250px src="https://gnaix92.github.io/blog_images/multi/18.png" style="display:inline-block"/>
<img width=700px height=250px src="https://gnaix92.github.io/blog_images/multi/19.png" style="display:inline-block"/>

###0x03 APK多开制作流程
　　上面主要是介绍了多开的原理，制作的过程是基于拿到源码的情况下。但是通过了解我们其实可以通过一些工具很容易（其实很没这么简单）的破解市场上的 APK。

####利用 apktool 反编译

　　利用 apktool 对 APK 进行反编译，这里以上面的 Eclipse 项目为例：

```shell
apktool d multi_demo.apk -o output
```
apktool 工具在之前的一篇有提到过，里面有提供下载地址：
[Android程序逆向分析](https://gnaix92.github.io/2016/03/06/android-reverse-engineering/)

####修改 AndroidManifest.xml
　　将 package 属性从 `com.example.multi_demo` 改为 `com.example.multi_demo_2`
因为 Activity 的包名也是 `com.example.multi_demo` 所以必须也要把 Activity 的 `android:name` 属性改为: `com.example.multi_demo.MainActivity` 不然会造成找不到该 Activity。
<img width=700px height=250px src="https://gnaix92.github.io/blog_images/multi/20.png" style="display:inline-block"/>

####重新编译
　　修改后就可以重新编译 APK 了，还是用 apktool：

```shell
apktool b output -o multi-demo_2.apk
```

####签名
　　apktool编译的apk并没有签名不能直接安装需要添加签名，用signapk.jar工具对apk进行签名。除了signapk.jar 还需要testkey.x509.pem testkey.pk8两个文件。工具可以在[/tools/auto-sign](https://github.com/gnaix92/crack/tree/master/tools/auto-sign)中找到。运行命令：

```shell
java -jar signapk.jar testkey.x509.pem testkey.pk8 multi-demo_2.apk sign.apk
```

　　签名后生成的sign.apk 就可以与原来的 multi-demo.apk 安装在同一台机子下了。

<font color="red">PS: </font>     
　　这种方式只能对部分没有做过加固处理和没有进行反二次打包处理的APK，还是很水的。    
　　另外 LBE 有个双开应用叫**平行空间**，基本可以实现所有应用的双开操作。不同于上面的修改包名操作，他们更像是为 APK 套了一层外壳，可以试着去获取被双开的应用包名，会发现与原来包名是一致的。    
　　但是其实也可以识别的，你去获取应用的绝对路径就知道了。