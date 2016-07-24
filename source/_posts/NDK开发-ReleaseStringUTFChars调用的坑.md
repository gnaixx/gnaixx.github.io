title: "NDK开发-ReleaseStringUTFChars调用的坑"
date: 2016-06-28 18:28:01
categories: android
tags: [android, ndk, jni]
toc: true
description: 在开发中部分关键代码是在 NDK 中处理的，但是前段时间我们在线上日志中发现有少量的日志报错。通过排查我们发现问题出现在 NDK 的 `ReleaseStringUTFChars` 函数上。虽然找到了解决办法，但是我还是没有分析出具体的问题在哪。

---

##0x00 问题描述
出于安全考虑我们会把部分关键的代码用 NKD 开发，加大被逆向的难度。但是可能是自己水平还不够，实际开发中会发现其实 NDK 开发还是有很多坑。     
出现问题的函数处理流程其实很简单，接收 JAVA String 变量，对其进行简单的处理，然后返回。      
示例代码：

```c++
NIEXPORT jstring JNICALL Java_cn_sdk_NativeEncode_getHello
        (JNIEnv *env, jclass object, jstring j_data, jstring j_key) {

	 //提取data key
    const char *c_data = env->GetStringUTFChars(j_data, 0);
    const char *c_key = env->GetStringUTFChars(j_key, 0);

    //计算长度
    int len_data = strlen(c_data);
    int len_key = strlen(c_key);
    //拷贝到cc_data cc_key
    char cc_data[len_data];
    char cc_key[len_key];
    strcpy(cc_data, c_data);
    strcpy(cc_key, c_key);

	 //是否字符串
    env->ReleaseStringUTFChars(j_data, c_data);
    env->ReleaseStringUTFChars(j_key, c_key);

    char result[1024];
    //数据处理

    return env->NewStringUTF(result);
}

```
其中很诡异的一点是：相同的data, 当我修改 key 时，有时候正常了。有时候 data 的一个字符的 `ASCII` 码是 **0**，这个就是导致失败的原因。

##0x01 代码分析
一开始我以为是数据处理部分代码有问题，但是经过多次的排查可以证实该部分代码没有问题。后来想是不是自己的代码在内存处理上有问题，导致了内存溢出或者野指针的情况。所以就找了一份 google的 NDK 开发 example。发现一般都是在 return 之前调用 `ReleaseStringUTF` 方法，而我在 copy 后就直接释放了。    

那么就有必要分析一下 `ReleaseStringUTF` 这个函数了，先看一下该函数的解释：

> 在调用 GetStringUTFChars 函数从 JVM 内部获取一个字符串之后，JVM 内部会分配一块新的内存，用于存储源字符串的拷贝，以便本地代码访问和修改。即然有内存分配，用完之后马上释放是一个编程的好习惯。通过调用ReleaseStringUTFChars 函数通知 JVM 这块内存已经不使用了，你可以清除了。注意：这两个函数是配对使用的，用了 GetXXX 就必须调用 ReleaseXXX，而且这两个函数的命名也有规律，除了前面的 Get 和 Release 之外，后面的都一样。

再看另一个相关的函数 `strcpy(src, dest)` 的解释：

>C语言标准库函数strcpy，把从src地址开始且含有'\0'结束符的字符串复制到以dest开始的地址空间。

所以代码运行的时候应该是先分配了一块新的内存地址保存 JAVA 变量的值，然后 `strcopy` 拷贝新内存的值 而不是地址，这时候释放新分配的内存，实际上不应该影响到 `strcopy(str, dest)` 中的 dest 的值。但是不幸，就是不行。


##0x02 解决方法

后来我就把`ReleaseStringUTF`放到数据处理结束后 return 前调用。卧槽还真就好了，也幸好线上拉下的错误日志可以复现 BUG 情况，所以就特么的修复了。

```c++
JNIEXPORT jstring JNICALL Java_cn_sdk_NativeEncode_getHello
        (JNIEnv *env, jclass object, jstring j_data) {

    const char *c_data = env->GetStringUTFChars(j_data, 0);
    const char *c_key = env->GetStringUTFChars(j_key, 0);

    int len_data = strlen(c_data);
    int len_key = strlen(c_key);
    char cc_data[len_data];
    char cc_key[len_key];

    char result[1024];
    //数据处理

    env->ReleaseStringUTFChars(j_data, c_data);
    return env->NewStringUTF(result);
}
```

##0x03 总结
其实我还是很懵逼，为什么我一开始写的会出现 bug 并且不是必现的，只有在极少数情况下才出现。希望有大神给我解释一下，解决我这一脸懵逼。