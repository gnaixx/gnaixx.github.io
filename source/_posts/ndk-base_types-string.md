title: "NDK开发 - JNI基本数据和字符串处理"
date: 2016-04-06 22:50:03
categories: android
tags: [android, ndk, jni]
toc: true
description: 第一篇搭建环境，第二篇了解开发流程，第三篇扯了一些理论，这一篇是时候展现真正的技术了，主要是写一些 JNI 的基本类型数据处理以及字符串的处理，字符串是最经常涉及的数据类型。

---

　　JNI 开发中主要涉及基本数据的处理，引用数据的处理，数组的处理。String 作为比较常用的引用数据类型 JNI 也提供了特有的的方法处理。在处理应用类型的时候需要注意内存的释放，不然很可能造成内存溢出。C/C++ 中的内存不能依靠 Java 的 GC 线程管理，下一篇会介绍 JNI 的引用类型。    
　　涉及的代码地址：[https://github.com/gnaix92/as-ndk](https://github.com/gnaix92/as-ndk)

###0x00 C和C++ 函数实现的比较
　　唯一的差异在于用来访问 JNI 函数的方法。在 C 中，JNI 函数调用由`(*env)->`作前缀，目的是为了取出函数指针所引用的值。在 C++ 中，JNIEnv 类拥有处理函数指针查找的内联成员函数。下面将说明这个细微的差异，其中，这两行代码访问同一函数，但每种语言都有各自的语法。    
C 语法： `jsize len = (*env)->GetArrayLength(env,array);`   
C++ 语法： `jsize len =env->GetArrayLength(array);`

　　下面涉及的代码都是用 C++ 开发的。所以看到网上用 C 开发的 JNI 也不需要奇怪。

###0x01 处理基本类型
　　JNI 中的基本类型和 Java 中的基本类型是一一对应的，这个在上一篇提到过。下面是 JNI 的基本类型定义：    

``` c++
typedef unsigned char   jboolean;  
typedef unsigned short  jchar;  
typedef short           jshort;  
typedef float           jfloat;  
typedef double          jdouble;  
typedef int             jint;  
#ifdef _LP64 /* 64-bit Solaris */  
typedef long            jlong;  
#else  
typedef long long       jlong;  
#endif  
typedef signed char     jbyte;
```
　　基本类型很容易理解，就是对 C/C++ 中的基本类型用 typedef 重新定义了一个新的名字，在 JNI 中可以直接访问。    

```c++
JNIEXPORT jint JNICALL Java_com_example_gnaix_ndk_NativeMethod_getInt
        (JNIEnv *env, jclass object, jint num)
{
    int len;
    char buf[1024];
    __system_property_get("ro.serialno", buf);
    LOGD("serialno : %s", buf);
    len = strlen(buf);
    return num + len;
}
```

###0x02 处理字符串

　　JNI 把 Java 中所有对象当作一个 C 指针传递到本地方法中，这个指针指向 JVM 中的内部数据结构，而内部的数据结构在内存中的存储方式是不可见得。只能从 JNIEnv 指针指向的函数表中选择合适的 JNI 函数来操作 JVM 中的数据结构。    
　　String 在 Java 是一个引用类型，所以在本地代码中只能通过GetStringUTFChars 这样的 JNI 函数来访问字符串的内容。
先看一个例子：

```c++
JNIEXPORT jstring JNICALL Java_com_example_gnaix_ndk_NativeMethod_getString
        (JNIEnv *env, jclass object, jstring str){

    //1. 将unicode编码的java字符串转换成C风格字符串
    const char *buf_name = env->GetStringUTFChars(str, 0);
    if(buf_name == NULL){
        return NULL;
    }
    int len = strlen(buf_name);
    char n_name[len];
    strcpy(n_name, buf_name);

    //2. 释放内存
    env->ReleaseStringUTFChars(str, buf_name);

    //3. 处理 n_name="ro.serialno"
    char buf[1024];
    __system_property_get(n_name, buf);
    LOGD("serialno : %s", buf);

    //4. 去掉尾部"\0"
    int len_buf = strlen(buf);
    string result(buf, len_buf);

    return env->NewStringUTF(result.c_str());
}
```
运行结果如下：
<img width=700px height=100px src="https://gnaix92.github.io/blog_images/ndk/4.png" style="display:inline-block"/>

####访问字符串
　　`getString` 函数接收一个 jstring 类型的参数 text，但是 jstring 类型是指向 JVM 内部的一个字符串，和 C/C++ 风格的字符串类型 char* 不同，所以在 JNI 中不能把 jstring 当做普通的 C/C++ 字符串一样来使用，必须使用 JNI 的函数来访问 JVM 内部的字符串数据结构。    
`const char* GetStringUTFChars(jstring string, jboolean* isCopy)`   
参数说明： 
    
- string：Java 传递给native代码的字符串指针
- isCopy： 取值 `JNI_TRUE` 和 `JNI_FALSE`，如果值为 `JNI_TRUE`，表示返回 JVM 内部源字符串的一份拷贝，并为新产生的字符串分配内存空间。如果值为 `JNI_FALSE`，表示返回 JVM 内部源字符串的指针，意味着可以通过指针修改源字符串的内容，不推荐这么做，因为这样做就打破了 Java 字符串不能修改的规定。但我们在开发当中，并不关心这个值是多少，通常情况下这个参数填 NULL 即可。   
 
　　因为 Java 默认使用 Unicode 编码，而 C/C++ 默认使用 UTF 编码，所以在本地代码中操作字符串的时候，必须使用合适的 JNI 函数把 jstring 转换成 C 风格的字符串。JNI 支持字符串在 Unicode 和 UTF-8 两种编码之间转换，`GetStringUTFChars` 可以把一个 jstring 指针（指向 JVM 内部的 Unicode 字符序列）转换成一个UTF-8 格式的 C 字符串。在上例中 `getString` 函数中我们通过 `GetStringUTFChars` 正确取得了 JVM 内部的字符串内容。

####异常检查
　　调用完 `GetStringUTFChars` 之后不要忘记安全检查，因为 JVM 需要为新诞生的字符串分配内存空间，当内存空间不够分配的时候，会导致调用失败，失败后 `GetStringUTFChars` 会返回 NULL，并抛出一个 `OutOfMemoryError` 异常。JNI 的异常和 Java 中的异常处理流程是不一样的，Java 遇到异常如果没有捕获，程序会立即停止运行。而 JNI 遇到未决的异常不会改变程序的运行流程，也就是程序会继续往下走，这样后面针对这个字符串的所有操作都是非常危险的，因此，我们需要用 return 语句跳过后面的代码，并立即结束当前方法。

####释放字符串
　　在调用 `GetStringUTFChars` 函数从 JVM 内部获取一个字符串之后，JVM 内部会分配一块新的内存，用于存储源字符串的拷贝，以便本地代码访问和修改。即然有内存分配，用完之后马上释放是一个编程的好习惯。通过调用 `ReleaseStringUTFChars` 函数通知 JVM 这块内存已经不使用了，你可以清除了。注意：这两个函数是配对使用的，用了 GetXXX 就必须调用 ReleaseXXX，而且这两个函数的命名也有规律，除了前面的 Get 和 Release 之外，后面的都一样。

####创建字符串
　　通过调用 `NewStringUTF` 函数，会构建一个新的 `java.lang.String` 字符串对象。这个新创建的字符串会自动转换成 Java 支持的 Unicode 编码。如果 JVM 不能为构造 `java.lang.String` 分配足够的内存，`NewStringUTF` 会抛出一个 `OutOfMemoryError` 异常，并返回 NULL。在这个例子中我们不必检查它的返回值，如果 `NewStringUTF` 创建 `java.lang.String` 失败，`OutOfMemoryError` 这个异常会被在 Java 方法中抛出。如果 `NewStringUTF` 创建 `java.lang.String` 成功，则返回一个 JNI 引用，这个引用指向新创建的`java.lang.String` 对象。

####其他字符串处理函数
　　下面这些是在编程中可能会用到的方法。

#####GetStringChar和ReleaseStringChars
　　这对函数和 `Get/ReleaseStringUTFChars` 函数功能差不多，用于获取和释放以 Unicode 格式编码的字符串。后者是用于获取和释放 UTF-8 编码的字符串。

#####GetStringLength
　　由于 UTF-8 编码的字符串以'\0'结尾，而 Unicode 字符串不是。如果想获取一个指向 Unicode 编码的 jstring 字符串长度，在 JNI 中可通过这个函数获取。

#####GetStringUTFLength
　　获取 UTF-8 编码字符串的长度，也可以通过标准 C 函数 strlen 获取。