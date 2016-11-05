title: "APK反逆向之二：四种基本加固方式"
date: 2016-10-15 11:28:37
categories: android
tags: [reserve, ndk, proguard, sign, dex, DexClassLoader]
toc: true
description: 近些年来移动 APP 数量呈现爆炸式的增长，黑产也从原来的PC端移到了移动端，伴随而来的逆向攻击手段也越来越高明。本篇章主要介绍应用加固的最基础的四种方式：1.proguard 混淆 2.签名比对验证 3.ndk 编译 .so 动态库 4.代码动态加载

---

## 0x00 简介

应该大多数开发者都不会关注应用会不逆向破解，而且现在有第三方厂商提供免费的加固方案，所以 apk 应用的安全性就全部依赖于第三方。但是如果第三方加固方案被破解那么 apk 就陷于被动，所以我们也可以通过一些手段来加固应用本身逻辑，或者数据的加密。

最常见的加固就是 proguard 混淆，通过混淆可以加大逻辑分析的难度。对于数据的加密通过 ndk 开发动态库基本上可以防止大部分的破解者，逆向 native 跟逆向 java 不是一个级别的。

下面介绍的四种加固方式成本比较低，开发者可以很快开发完成，并且也不会影响应用的兼容性。

## 0x01 proguard混淆

### 简介
ProGuard 是一个 SourceForge 上知名的开源项目。官网网址是：[http://proguard.sourceforge.net/](http://proguard.sourceforge.net/)。

java 编译后的字节码很容易被反编译，基本上 java 编译器都可以比如 eclipse、idea，还有用的比较多的 JD-GUI。都可以将 java 字节码转化为 java 源码。ProGuard的主要作用就是混淆。当然它还能对字节码进行缩减体积、优化等，但那些对于我们来说都算是次要的功能。

>ProGuard is a free Java class file shrinker, optimizer, obfuscator, and preverifier. It detects and removes unused classes, fields, methods, and attributes. It optimizes bytecode and removes unused instructions. It renames the remaining classes, fields, and methods using short meaningless names. The resulting applications and libraries are smaller, faster, and a bit better hardened against reverse engineering.

### Android 配置
Android 2.3 以后 SDK 中已经默认支持 ProGuard 混淆，具体配置文件在 `sdk-root/tools/proguard/proguard-android.txt`。 
   
SDK 内置 ProGuard 混淆文件：

```bash
# This is a configuration file for ProGuard.
# http://proguard.sourceforge.net/index.html#manual/usage.html

-dontusemixedcaseclassnames
-dontskipnonpubliclibraryclasses
-verbose

# Optimization is turned off by default. Dex does not like code run
# through the ProGuard optimize and preverify steps (and performs some
# of these optimizations on its own).
-dontoptimize
-dontpreverify
# Note that if you want to enable optimization, you cannot just
# include optimization flags in your own project configuration file;
# instead you will need to point to the
# "proguard-android-optimize.txt" file instead of this one from your
# project.properties file.

-keepattributes *Annotation*
-keep public class com.google.vending.licensing.ILicensingService
-keep public class com.android.vending.licensing.ILicensingService

# For native methods, see http://proguard.sourceforge.net/manual/examples.html#native
-keepclasseswithmembernames class * {
    native <methods>;
}

# keep setters in Views so that animations can still work.
# see http://proguard.sourceforge.net/manual/examples.html#beans
-keepclassmembers public class * extends android.view.View {
   void set*(***);
   *** get*();
}

# We want to keep methods in Activity that could be used in the XML attribute onClick
-keepclassmembers class * extends android.app.Activity {
   public void *(android.view.View);
}

# For enumeration classes, see http://proguard.sourceforge.net/manual/examples.html#enumerations
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

-keepclassmembers class * implements android.os.Parcelable {
  public static final android.os.Parcelable$Creator CREATOR;
}

-keepclassmembers class **.R$* {
    public static <fields>;
}

# The support library contains references to newer platform versions.
# Don't warn about those in case this app is linking against an older
# platform version.  We know about them, and they are safe.
-dontwarn android.support.**

# Understand the @Keep support annotation.
-keep class android.support.annotation.Keep

-keep @android.support.annotation.Keep class * {*;}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <methods>;
}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <fields>;
}

-keepclasseswithmembers class * {
    @android.support.annotation.Keep <init>(...);
}
```

在 Android Studio 中构建新项目时会在每个 module 下生成 `proguard-rules.pro` 这里可以配置一些自定义的 ProGuard 规则。在 build.gradle 中配置是否启动混淆配置, 一般情况下编译 release 版本会启用混淆，debug 版本关闭：

```java
buildTypes {
    release {
        signingConfig signingConfigs.tdSignConf
        minifyEnabled true
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
    debug{
        signingConfig signingConfigs.tdSignConf
        minifyEnabled false
    }
}
```

### 生成文件
在Android Studio 中启用 ProGuard 混淆会在`/app/build/outputs/mapping/release`下生成下面四个文件：

- mapping.txt —> 表示混淆前后代码的对照表
- dump.txt  —> 描述apk内所有class文件的内部结构
- seeds.txt —> 列出了没有被混淆的类和成员
- usage.txt —> 列出了源代码中被删除在apk中不存在的代码

mapping.txt 文件非常重要。如果你的代码混淆后会产生bug的话，log提示中是混淆后的代码，希望定位到源代码的话就可以根据mapping.txt反推。每次发布都要保留它方便该版本出现问题时调出日志进行排查，它可以根据版本号或是发布时间命名来保存或是放进代码版本控制中。

如果是手动 ProGuard 混淆可以在配置文件中添加 `-printmapping`选项。

### 异常堆栈还原
混淆后代码变量、函数名、类名、都已经变成了随机的字母，crash信息也就看不错在什么地方。所以就需要上面提到 mapping.txt 保存了混淆前后的代码对照表。

例如有个错误日志：

```java
Caused by: java.lang.NullPointerException
at cc.gnaixx.be.u(Unknown Source)
at cc.gnaixx.at.v(Unknown Source)
at cc.gnaixx.at.d(Unknown Source)
at cc.gnaixx.av.onReceive(Unknown Source)
```
这个日志中我们根本无法排查错误在哪一行，但是 ProGuard 提供了一个还原工具，工具在`proguard/bin/retrace.sh`。
执行:

```bash
#crash.txt 为错误栈文件
retrace.sh -verbose mapping.txt crash.txt
```

执行后的效果：

```java
Caused by: java.lang.NullPointerException
at cc.gnaixx.UtilTelephony.boolean is800MhzNetwork()(Unknown Source)
at cc.gnaixx.ServiceDetectLte.void checkAndAlertUserIf800MhzConnected()(Unknown Source)
at cc.gnaixx.ServiceDetectLte.void startLocalBroadcastReceiver()(Unknown Source)
at cc.gnaixx.ServiceDetectLte$2.void onReceive(android.content.Context,android.content.Intent)(Unknown Source)
```
这里需要注意一点如果你想查单个错误信息，必须在错误信息前加 `at`, 不然无法找出错误位置。

### 配置规则

**基本选项**

```bash
-include {filename}                   从给定的文件中读取配置参数
-basedirectory {directoryname}        指定基础目录为以后相对的档案名称 
-injars {class_path}                  指定要处理的应用程序jar,war,ear和目录   
-outjars {class_path}                 指定处理完后要输出的jar,war,ear和目录的名称   
-libraryjars {classpath}              指定要处理的应用程序jar,war,ear和目录所需要的程序库文件   
-dontskipnonpubliclibraryclasses      指定不去忽略非公共的库类。   
-dontskipnonpubliclibraryclassmembers 指定不去忽略包可见的库类的成员。 
```
**保留选项**

```bash
-keep {Modifier} {class_specification}                保护指定的类文件和类的成员   
-keepclassmembers {modifier} {class_specification}    保护指定类的成员，如果此类受到保护他们会保护的更好  
-keepclasseswithmembers {class_specification}         保护指定的类和类的成员，但条件是所有指定的类和类成员是要存在。   
-keepnames {class_specification}                      保护指定的类和类的成员的名称（如果他们不会压缩步骤中删除）   
-keepclassmembernames {class_specification}           保护指定的类的成员的名称（如果他们不会压缩步骤中删除）   
-keepclasseswithmembernames {class_specification}     保护指定的类和类的成员的名称，如果所有指定的类成员出席（在压缩步骤之后）   
-printseeds {filename}                                列出类和类的成员-keep选项的清单，标准输出到给定的文件   
```
**压缩**

```bash
-dontshrink    不压缩输入的类文件   
-printusage {filename}   
-whyareyoukeeping {class_specification} 
```

**优化**

```bash
-dontoptimize                                 不优化输入的类文件   
-assumenosideeffects {class_specification}    优化时假设指定的方法，没有任何副作用   
-allowaccessmodification                      优化时允许访问并修改有修饰符的类和类的成员  
```
**混淆**

```bash
-dontobfuscate                             不混淆输入的类文件   
-printmapping {filename}                   混淆前后代码的对照表
-applymapping {filename}                   重用映射增加混淆   
-obfuscationdictionary {filename}          使用给定文件中的关键字作为要混淆方法的名称   
-overloadaggressively                      混淆时应用侵入式重载   
-useuniqueclassmembernames                 确定统一的混淆类的成员名称来增加混淆   
-flattenpackagehierarchy {package_name}    重新包装所有重命名的包并放在给定的单一包中   
-repackageclass {package_name}             重新包装所有重命名的类文件中放在给定的单一包中   
-dontusemixedcaseclassnames                混淆时不会产生形形色色的类名   
-keepattributes {attribute_name,...}       保护给定的可选属性，例如LineNumberTable, LocalVariableTable, SourceFile, Deprecated, Synthetic, Signature, InnerClasses.   
-renamesourcefileattribute {string}        设置源文件中给定的字符串常量  
```

### 使用总结

下面这些规则主要是自己在使用过程中遇到过的问题

```bash
# 忽略指点错误，减少打印日志，因为我是针对jar做混淆，打印一堆没用日志有点烦
-dontnote java.**, javax.**, org.**

# 日志打印类名重新定义为"ProGuard"字符串
-renamesourcefileattribute ProGuard

# 保留源文件名,保留行号,保留annotation
-keepattributes SourceFile, LineNumberTable, *Annotation*

# 保持枚举 enum 类不被混淆
-keepclassmembers enum * {                    
    public static **[] values();  
    public static ** valueOf(java.lang.String);  
} 

# 保存内部类不被混淆
-keep class cc.gnaixx.TestClass$* {*;} 

```

## 0x02 签名比对验证

### 简介
Android APK的发布是需要签名的。签名机制在Android应用和框架中有着十分重要的作用。

例如，Android系统禁止更新安装签名不一致的APK；如果应用需要使用system权限，必须保证APK签名与Framework签名一致，等等。要破解一个APK，必然需要重新对APK进行签名。而这个签名，一般情况无法再与APK原先的签名保持一致。（除非APK原作者的私钥泄漏）简单地说，签名机制标明了APK的发行机构。

### Android签名机制
 为了说明APK签名比对对软件安全的有效性，我们有必要了解一下Android APK的签名机制。为了更易于大家理解，我们从Auto-Sign工具的一条批处理命令说起。

要签名一个没有签名过的APK，可以使用一个叫作auto-sign的工具。auto-sign工具实现签名功能的命令主要是这一行命令：

```bash
java -jar signapk.jar testkey.x509.pem testkey.pk8 update.apk update_signed.apk
```
这条命令的意义是：通过signapk.jar这个可执行jar包，以“testkey.x509.pem”这个公钥文件和“testkey.pk8”这个私钥文件对“update.apk”进行签名，签名后的文件保存为“update_signed.apk”。

对于此处所使用的私钥和公钥的生成方式，这里就不做进一步介绍了。这方面的资料大家可以找到很多。我们这里要讲的是signapk.jar到底做了什么。

signapk.jar是Android源码包中的一个签名工具。由于Android是个开源项目，所以，很高兴地，我们可以直接找到signapk.jar的源码！路径为/build/tools/signapk/SignApk.java。

对比一个没有签名的APK和一个签名好的APK，我们会发现，签名好的APK包中多了一个叫做META-INF的文件夹。里面有三个文件，分别名为MANIFEST.MF、CERT.SF和CERT.RSA。signapk.jar就是生成了这几个文件（其他文件没有任何改变。因此我们可以很容易去掉原有签名信息）。

通过阅读signapk源码，我们可以理清签名APK包的整个过程。

**1、 生成MANIFEST.MF文件：**

程序遍历update.apk包中的所有文件(entry)，对非文件夹非签名文件的文件，逐个生成SHA1的数字签名信息，再用Base64进行编码。具体代码见这个方法：

```java
private static Manifest addDigestsToManifest(JarFile jar)

//关键代码如下：
for (JarEntry entry: byName.values()) {
  String name = entry.getName();
  if (!entry.isDirectory() && !name.equals(JarFile.MANIFEST_NAME) &&
      !name.equals(CERT_SF_NAME) && !name.equals(CERT_RSA_NAME) &&
             (stripPattern == null ||!stripPattern.matcher(name).matches())) {
                
    InputStream data = jar.getInputStream(entry);
                while ((num = data.read(buffer)) > 0) {
                    md.update(buffer, 0, num);
                }
                Attributes attr = null;
                if (input != null) attr = input.getAttributes(name);
                attr = attr != null ? new Attributes(attr) : new Attributes();
                attr.putValue("SHA1-Digest", base64.encode(md.digest()));
                output.getEntries().put(name, attr);
    }
}
```
之后将生成的签名写入MANIFEST.MF文件。关键代码如下：

```java
Manifest manifest = addDigestsToManifest(inputJar);
je = new JarEntry(JarFile.MANIFEST_NAME);
je.setTime(timestamp);
outputJar.putNextEntry(je);
manifest.write(outputJar);
```
这里简单介绍下SHA1数字签名。简单地说，它就是一种安全哈希算法，类似于MD5算法。它把任意长度的输入，通过散列算法变成固定长度的输出（这里我们称作“摘要信息”）。你不能仅通过这个摘要信息复原原来的信息。另外，它保证不同信息的摘要信息彼此不同。因此，如果你改变了apk包中的文件，那么在apk安装校验时，改变后的文件摘要信息与MANIFEST.MF的检验信息不同，于是程序就不能成功安装。

**2、生成CERT.SF文件：**

对前一步生成的Manifest，使用SHA1-RSA算法，用私钥进行签名。关键代码如下：

```java
Signature signature = Signature.getInstance("SHA1withRSA");
signature.initSign(privateKey);
je = new JarEntry(CERT_SF_NAME);
je.setTime(timestamp);
outputJar.putNextEntry(je);
writeSignatureFile(manifest,
new SignatureOutputStream(outputJar, signature));
```
RSA是一种非对称加密算法。用私钥通过RSA算法对摘要信息进行加密。在安装时只能使用公钥才能解密它。解密之后，将它与未加密的摘要信息进行对比，如果相符，则表明内容没有被异常修改。

**3、生成CERT.RSA文件：**

生成MANIFEST.MF没有使用密钥信息，生成CERT.SF文件使用了私钥文件。那么我们可以很容易猜测到，CERT.RSA文件的生成肯定和公钥相关。

CERT.RSA文件中保存了公钥、所采用的加密算法等信息。核心代码如下：

```java
je = new JarEntry(CERT_RSA_NAME);
je.setTime(timestamp);
outputJar.putNextEntry(je);
writeSignatureBlock(signature, publicKey, outputJar);

//其中writeSignatureBlock的代码如下：
private static void writeSignatureBlock(
      Signature signature, X509Certificate publicKey, OutputStream out)
          throws IOException, GeneralSecurityException {
            SignerInfo signerInfo = new SignerInfo(
                new X500Name(publicKey.getIssuerX500Principal().getName()),
                publicKey.getSerialNumber(),
                AlgorithmId.get("SHA1"),
                AlgorithmId.get("RSA"),
                signature.sign());

        PKCS7 pkcs7 = new PKCS7(
          new AlgorithmId[] { AlgorithmId.get("SHA1") },
            new ContentInfo(ContentInfo.DATA_OID, null),
            new X509Certificate[] { publicKey },
            new SignerInfo[] { signerInfo });

        pkcs7.encodeSignedData(out);
}
```
分析完APK包的签名流程，我们可以清楚地意识到：

1. Android签名机制其实是对APK包完整性和发布机构唯一性的一种校验机制。
2. Android签名机制不能阻止APK包被修改，但修改后的再签名无法与原先的签名保持一致。（拥有私钥的情况除外）。
3. APK包加密的公钥就打包在APK包内，且不同的私钥对应不同的公钥。换句话言之，不同的私钥签名的APK公钥也必不相同。所以我们可以根据公钥的对比，来判断私钥是否一致。

### APK签名比对的实现方式
具体的实现方式在之前的一篇博客中有进行了介绍，实现方式其实很简单：

[APK自我保护 - DEX/APK/证书校验](http://gnaixx.cc/2016/04/19/android-protect-dex_apk_cert_check/)


## 0x03 ndk 编译.so 动态库

### 简介
逆向 ndk 跟逆向 java 代码难度真不是在一个等级上的，ndk 开发除了安全性相对 java 开发高很多，而且在运行效率也高很多。

所以 NDK 开发相对于 JAVA 开发的优势：

1.代码的保护。由于apk的java层代码很容易被反编译，而C/C++库反汇难度较大。

2.可以方便地使用现存的开源库。大部分现存的开源库都是用C/C++代码编写的。

3.提高程序的执行效率。将要求高性能的应用逻辑使用C开发，从而提高应用程序的执行效率。

4.便于移植。用C/C++写得库可以方便在其他的嵌入式平台上再次使用。

但是 NDK 的兼容性相对 JAVA 差很多，所以在进行 NDK 开发的时候兼容性是一个很大的问题。

### 教程
之前写了几篇关于 NDK 开发的文章基本覆盖了 NDK 开发的所有内容了。

- [NDK开发 - Android Studio环境搭建](http://gnaixx.cc/2016/03/07/ndk-android_studio-dev-env/)
- [NDK开发 - JNI开发流程](http://gnaixx.cc/2016/03/22/ndk-precess/)
- [NDK开发 - JNI数据类型与Java数据类型映射关系](http://gnaixx.cc/2016/03/30/ndk-jni_java-mapping/)
- [NDK开发 - JNI基本数据和字符串处理](http://gnaixx.cc/2016/04/06/ndk-base_types-string/)
- [NDK开发 - JNI数组数据处理](http://gnaixx.cc/2016/04/07/ndk-array/)
- [NDK开发 - C/C++ 访问 Java 变量和方法](http://gnaixx.cc/2016/04/08/ndk-invoke-java-method_field/)
- [NDK开发 - JNI 局部引用、全局引用和弱全局引用](http://gnaixx.cc/2016/04/08/ndk-local_global_weak-reference/)

这些教程都是基于 experimental 版本的 gradle 进行开发，但是这种版本的配置相对于普通版本的 gradle 有很多不同。并且创建一个新的 module 都需要更新 build.gradle 配置非常麻烦。

最近 Android Studio 更新到了2.2 版本，可以支持两种方式的 NDK开发：

**1. ndk-build 开发**

这种方式非常简单只需要在 build.gradle 添加 ndk 编译配置：

```java
externalNativeBuild {
     ndkBuild {
         path 'src/main/jni/Android.mk'
     }
 }
```
**2. cmake 开发**

先贴出官方文档，以后再介绍。

官网文档：
[Add C and C++ Code to Your Project](https://developer.android.com/studio/projects/add-native-code.html#existing-project)

## 0x04 代码动态加载

### 简介
这篇文章是在看雪上看到的对动态加载的原理介绍的比较清楚，大多数应用也是采用这种方式对 dex 文件进行加密处理。
    
熟悉Java技术的朋友，可能意识到，我们需要使用类加载器灵活的加载执行的类。这在Java里已经算是一项比较成熟的技术了，但是在Android中，我们大多数人都还非常陌生。

### 类加载机制       

Dalvik虚拟机如同其他Java虚拟机一样，在运行程序时首先需要将对应的类加载到内存中。而在Java标准的虚拟机中，类加载可以从class文件中读取，也可以是其他形式的二进制流。因此，我们常常利用这一点，在程序运行时手动加载Class，从而达到代码动态加载执行的目的。

然而Dalvik虚拟机毕竟不算是标准的Java虚拟机，因此在类加载机制上，它们有相同的地方，也有不同之处。我们必须区别对待。

例如，在使用标准Java虚拟机时，我们经常自定义继承自ClassLoader的类加载器。然后通过defineClass方法来从一个二进制流中加载Class。然而，这在Android里是行不通的，大家就没必要走弯路了。参看源码我们知道，Android中ClassLoader的defineClass方法具体是调用VMClassLoader的defineClass本地静态方法。而这个本地方法除了抛出一个“UnsupportedOperationException”之外，什么都没做，甚至连返回值都为空。

```java
static void Dalvik_java_lang_VMClassLoader_defineClass(const u4* args, JValue* pResult)
{

    Object* loader = (Object*) args[0];
    StringObject* nameObj = (StringObject*) args[1];
    const u1* data = (const u1*) args[2];
    int offset = args[3];
    int len = args[4];

    Object* pd = (Object*) args[5];
    char* name = NULL;

    name = dvmCreateCstrFromString(nameObj);
    LOGE("ERROR: defineClass(%p, %s, %p, %d, %d, %p)\n", loader, name, data, offset, len, pd);

    dvmThrowException("Ljava/lang/UnsupportedOperationException;", "can't load this type of class file");

    free(name);
    RETURN_VOID();
}
```
### Dalvik虚拟机类加载机制

那如果在Dalvik虚拟机里，ClassLoader不好使，我们如何实现动态加载类呢？Android为我们从ClassLoader派生出了两个类：DexClassLoader和PathClassLoader。其中需要特别说明的是PathClassLoader中一段被注释掉的代码：

```java
/**//* --this doesn't work in current version of Dalvik--

    if (data != null) {

        System.out.println("--- Found class " + name + " in zip[" + i + "] '" + mZips[i].getName() + "'");

        int dotIndex = name.lastIndexOf('.');
        if (dotIndex != -1) {

            String packageName = name.substring(0, dotIndex);

            synchronized (this) {

                Package packageObj = getPackage(packageName);

                if (packageObj == null) {
                    definePackage(packageName, null, null, null, null, null, null, null);
                }
            }
        }

        return defineClass(name, data, 0, data.length);
    }
*/
```

这从另一方面佐证了defineClass函数在Dalvik虚拟机里确实是被阉割了。而在这两个继承自ClassLoader的类加载器，本质上是重载了ClassLoader的findClass方法。在执行loadClass时，我们可以参看ClassLoader部分源码：

```java
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {

    Class<?> clazz = findLoadedClass(className);

    if (clazz == null) {

        try {
            clazz = parent.loadClass(className, false);
        } catch (ClassNotFoundException e) {
            // Don't want to see this.
        }

        if (clazz == null) {
            clazz = findClass(className);
        }
    }

    return clazz;
}
```
因此DexClassLoader和PathClassLoader都属于符合双亲委派模型的类加载器（因为它们没有重载loadClass方法）。也就是说，它们在加载一个类之前，回去检查自己以及自己以上的类加载器是否已经加载了这个类。如果已经加载过了，就会直接将之返回，而不会重复加载。

DexClassLoader和PathClassLoader其实都是通过DexFile这个类来实现类加载的。这里需要顺便提一下的是，Dalvik虚拟机识别的是dex文件，而不是class文件。因此，我们供类加载的文件也只能是dex文件，或者包含有dex文件的.apk或.jar文件。

也许有人想到，既然DexFile可以直接加载类，那么我们为什么还要使用ClassLoader的子类呢？DexFile在加载类时，具体是调用成员方法loadClass或者loadClassBinaryName。其中loadClassBinaryName需要将包含包名的类名中的”.”转换为”/”。我们看一下loadClass代码就清楚了：

```java
public Class loadClass(String name, ClassLoader loader) {

        String slashName = name.replace('.', '/');
        return loadClassBinaryName(slashName, loader);
}
```

在这段代码前有一段注释，截取关键一部分就是说：If you are not calling this from a class loader, this is most likely not going to do what you want. Use {@link Class#forName(String)} instead. 这就是我们需要使用ClassLoader子类的原因。至于它是如何验证是否是在ClassLoader中调用此方法的，我没有研究，大家如果有兴趣可以继续深入下去。

有一个细节，可能大家不容易注意到。PathClassLoader是通过构造函数new DexFile(path)来产生DexFile对象的；而DexClassLoader则是通过其静态方法loadDex（path, outpath, 0）得到DexFile对象。这两者的区别在于DexClassLoader需要提供一个可写的outpath路径，用来释放.apk包或者.jar包中的dex文件。换个说法来说，就是PathClassLoader不能主动从zip包中释放出dex，因此只支持直接操作dex格式文件，或者已经安装的apk（因为已经安装的apk在cache中存在缓存的dex文件）。而DexClassLoader可以支持.apk、.jar和.dex文件，并且会在指定的outpath路径释放出dex文件。

另外，PathClassLoader在加载类时调用的是DexFile的loadClassBinaryName，而DexClassLoader调用的是loadClass。因此，在使用PathClassLoader时类全名需要用”/”替换”.”。

### 实际操作

这一部分比较简单，因此我就不赘言了。只是简单的说下。

可能使用到的工具都比较常规：javac、dx、eclipse等。其中dx工具最好是指明--no-strict，因为class文件的路径可能不匹配。

加载好类后，通常我们可以通过Java反射机制来使用这个类。但是这样效率相对不高，而且老用反射代码也比较复杂凌乱。更好的做法是定义一个interface，并将这个interface写进容器端。待加载的类，继承自这个interface，并且有一个参数为空的构造函数，以使我们能够通过Class的newInstance方法产生对象。然后将对象强制转换为interface对象，于是就可以直接调用成员方法了。


### 关于代码加密的一些设想

最初设想将dex文件加密，然后通过JNI将解密代码写在Native层。解密之后直接传上二进制流，再通过defineClass将类加载到内存中。

现在也可以这样做，但是由于不能直接使用defineClass，而必须传文件路径给dalvik虚拟机内核，因此解密后的文件需要写到磁盘上，增加了被破解的风险。

Dalvik虚拟机内核仅支持从dex文件加载类的方式是不灵活的，由于没有非常深入的研究内核，我不能确定是Dalvik虚拟机本身不支持还是Android在移植时将其阉割了。不过相信Dalvik或者是Android开源项目都正在向能够支持raw数据定义类方向努力。

我们可以在文档中看到Google说：Jar or APK file with "classes.dex". (May expand this to include "raw DEX" in the future.)；在Android的Dalvik源码中我们也能看到RawDexFile的身影（不过没有具体实现）。

在RawDexFile出来之前，我们都只能使用这种存在一定风险的加密方式。需要注意释放的dex文件路径及权限管理，另外，在加载完毕类之后，除非出于其他目的否则应该马上删除临时的解密文件

### 总结
目前的方式如果被识别后破解还是相对比较简单的，即使加密后的 dex 文件也需要在 load 前进行解密操作，并且需要把文件缓存在本地。所以只需要在合适的地方下个断点就可以把加密的 dex 拷贝下来。

但是我们可以通过其他方式对 dex 进行加密，比如隐藏函数。这样即使拿到了 dex 也无法通过逆向工具查看源码。

**参考文章：**

- [ProGuard 官方文档](http://proguard.sourceforge.net/)
- [Android 中 ProGuard 配置和总结](http://treesouth.github.io/2015/04/05/Android%E4%B8%ADProGuard%E6%B7%B7%E6%B7%86%E9%85%8D%E7%BD%AE%E5%92%8C%E6%80%BB%E7%BB%93/)
- [Android APK 签名比对](http://bbs.pediy.com/showthread.php?t=137500)
- [Android类动态加载技术](http://bbs.pediy.com/showthread.php?t=142256)


