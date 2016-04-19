title: "NDK开发-JNI 局部引用、全局引用和弱全局引用"
date: 2016-04-08 15:17:01
categories: android
tags: [android, ndk, jni]
toc: true
description: 看到一篇写的很不错的介绍JNI各种引用的文章 mark 了。

---
　　这篇文章比较偏理论，详细介绍了在编写本地代码时三种引用的使用场景和注意事项。可能看起来有点枯燥，但引用是在 JNI 中最容易出错的一个点，如果使用不当，容易使程序造成内存溢出，程序崩溃等现象。《Android JNI局部引用表溢出》这篇文章是一个 JNI 引用使用不当造成引用表溢出，最终导致程序崩溃的例子。建议看完这篇文章之后，再去看。

　　做 Java 的朋友都知道，在编码的过程当中，内存管理这一块完全是透明的。new 一个类的实例时，只知道创建完这个类的实例之后，会返回这个实例的一个引用，然后就可以拿着这个引用访问它的所有数据成员了（属性、方法）。完全不用管 JVM 内部是怎么实现的，如何为新创建的对象来申请内存，也不用管对象使用完之后内存是怎么释放的，只需知道有一个垃圾回器在帮忙管理这些事情就 OK 的了。有经验的朋友也许知道启动一个 Java 程序，如果没有手动创建其它线程，默认会有两个线程在跑，一个是 main 线程，另一个就是 GC 线程（负责将一些不再使用的对象回收）。如果你曾经是做 Java 的然后转去做 C++，会感觉很不习惯，在 C++ 中 new 一个对象，使用完了还要做一次 delete 操作，malloc 一次同样也要调用 free 来释放相应的内存，否则你的程序就会有内存泄露了。而且在 C/C++ 中内存还分栈空间和堆空间，其中局部变量、函数形参变量、for 中定义的临时变量所分配的内存空间都是存放在栈空间（而且还要注意大小的限制），用 new 和 malloc 申请的内存都存放在堆空间。但 C/C++ 里的内存管理还远远不止这些，这些只是最基础的内存管理常识。做 Java 的人听到这些肯定会偷乐了，咱写 Java 的时候这些都不用管，全都交给 GC 就万事无优了。手动管理内存虽然麻烦，而且需要特别细心，一不小心就有可能造成内存泄露和野指针访问等程序致命的问题，但凡事都有利弊，手动申请和释放内存对程序的掌握比较灵活，不会受到平台的限制。比如我们写Android程序的时候，内存使用就受Dalivk虚拟机的限制，从最初版本的16~24M，到后来的 32M 到 64M，可能随着以后移动设备物理内存的不大扩大，后面的 Android 版本内存限制可能也会随着提高。但在 C/C++ 这层，就完全不受虚拟机的限制了。比如要在 Android 中要存储一张超高清的图片，刚好这张图片的大小超过了 Dalivk 虚拟机对每个应用的内存大小限制，Java 此时就显得无能为力了，但在C/C++ 看来就是小菜一碟了，malloc(1024102450)。C/C++ 程序员得意的说道，Java 不是说是一门纯面象对象的语言吗，所以除了基本数据类型外，其它任何类型所创建的对象，JVM 所申请的内存都存在堆空间。上面提高到了 GC，是负责回收不再使用的对象，它的全称是 Garbage Collection，也就是所谓的垃圾回收。JVM 会在适当的时机触发 GC 操作，一旦进行 GC 操作，就会将一些不再使用的对象进行回收。那么哪些对象会被认为是不再使用，并且可以被回收的呢？我们来看下面二张图。（注：图摘自博主郭霖的《Android 最佳性能实践(二)——分析内存的使用情况》）

![http://gnaix92.github.io/blog_images/ndk/12.png](http://gnaix92.github.io/blog_images/ndk/12.png)

　　上图当中，每个蓝色的圆圈就代表一个内存当中的对象，而圆圈之间的箭头就是它们的引用关系。这些对象有些是处于活动状态的，而有些就已经不再被使用了。那么 GC 操作会从一个叫作 Roots 的对象开始检查，所有它可以访问到的对象就说明还在使用当中，应该进行保留，而其它的对象就表示已经不再被使用了，如下图所示：

![http://gnaix92.github.io/blog_images/ndk/13.png](http://gnaix92.github.io/blog_images/ndk/13.png)

　　可以看到，目前所有黄色的对象都处于活动状态，仍然会被系统继续保留，而蓝色的对象就会在 GC 操作当中被系统回收掉了，这就是 JVM 执行一次 GC 的简单流程。

　　上面说的废话好像有点多哈，下面进入正题。通过上面的讨论，大家都知道，如果一个 Java 对象没有被其它成员变量或静态变量所引用的话，就随时有可能会被 GC 回收掉。所以我们在编写本地代码时，要注意从 JVM 中获取到的引用在使用时被 GC 回收的可能性。由于本地代码不能直接通过引用操作 JVM 内部的数据结构，要进行这些操作必须调用相应的 JNI 接口来间接操作所引用的数据结构。JNI 提供了和 Java 相对应的引用类型，供本地代码配合 JNI 接口间接操作 JVM 内部的数据内容使用。如：jobject、jstring、jclass、jarray、jintArray 等。因为我们只通过 JNI 接口操作 JNI 提供的引用类型数据结构，而且每个 JVM 都实现了 JNI 规范相应的接口，所以我们不必担心特定 JVM 中对象的存储方式和内部数据结构等信息，我们只需要学习 JNI 中三种不同的引用即可。

　　由于 Java 程序运行在虚拟机中的这个特点，在 Java 中创建的对象、定义的变量和方法，内部对象的数据结构是怎么定义的，只有 JVM 自己知道。如果我们在 C/C++ 中想要访问 Java 中对象的属性和方法时，是不能够直接操作 JVM 内部 Java 对象的数据结构的。想要在 C/C++ 中正确的访问 Java 的数据结构，JVM 就必须有一套规则来约束 C/C++ 与 Java 互相访问的机制，所以才有了 JNI 规范，JNI 规范定义了一系列接口，任何实现了这套 JNI 接口的 Java 虚拟机，C/C++ 就可以通过调用这一系列接口来间接的访问 Java 中的数据结构。比如前面文章中学习到的常用 JNI 接口有：GetStringUTFChars（从 Java 虚拟机中获取一个字符串）、ReleaseStringUTFChars（释放从 JVM 中获取字符串所分配的内存空间）、NewStringUTF、GetArrayLength、GetFieldID、GetMethodID、FindClass 等。

###三种引用简介及区别
　　在 JNI 规范中定义了三种引用：局部引用（Local Reference）、全局引用（Global Reference）、弱全局引用（Weak Global Reference）。区别如下：

####局部引用
　　通过 NewLocalRef 和各种 JNI 接口创建（FindClass、NewObject、GetObjectClass和NewCharArray等）。会阻止 GC 回收所引用的对象，不在本地函数中跨函数使用，不能跨线前使用。函数返回后局部引用所引用的对象会被JVM 自动释放，或调用 DeleteLocalRef 释放。`(*env)->DeleteLocalRef(env,local_ref)`

```c++
jclass cls_string = (*env)->FindClass(env, "java/lang/String");
jcharArray charArr = (*env)->NewCharArray(env, len);
jstring str_obj = (*env)->NewObject(env, cls_string, cid_string, elemArray);
jstring str_obj_local_ref = (*env)->NewLocalRef(env,str_obj);   // 通过NewLocalRef函数创建
...
```

####全局引用
　　调用 NewGlobalRef 基于局部引用创建，会阻 GC 回收所引用的对象。可以跨方法、跨线程使用。JVM 不会自动释放，必须调用 DeleteGlobalRef 手动释放。`(*env)->DeleteGlobalRef(env,g_cls_string)`

```c++
static jclass g_cls_string;
void TestFunc(JNIEnv* env, jobject obj) {
    jclass cls_string = (*env)->FindClass(env, "java/lang/String");
    g_cls_string = (*env)->NewGlobalRef(env,cls_string);
}
```

####弱全局引用
　　调用 NewWeakGlobalRef 基于局部引用或全局引用创建，不会阻止 GC 回收所引用的对象，可以跨方法、跨线程使用。引用不会自动释放，在 JVM 认为应该回收它的时候（比如内存紧张的时候）进行回收而被释放。或调用DeleteWeakGlobalRef 手动释放。`(*env)->DeleteWeakGlobalRef(env,g_cls_string)`

```c++
static jclass g_cls_string;
void TestFunc(JNIEnv* env, jobject obj) {
    jclass cls_string = (*env)->FindClass(env, "java/lang/String");
    g_cls_string = (*env)->NewWeakGlobalRef(env,cls_string);
}
```

####局部引用

　　局部引用也称本地引用，通常是在函数中创建并使用。会阻止 GC 回收所引用的对象。比如，调用 NewObject 接口创建一个新的对象实例并返回一个对这个对象的局部引用。局部引用只有在创建它的本地方法返回前有效，本地方法返回到 Java 层之后，如果 Java 层没有对返回的局部引用使用的话，局部引用就会被 JVM 自动释放。你可能会为了提高程序的性能，在函数中将局部引用存储在静态变量中缓存起来，供下次调用时使用。这种方式是错误的，因为函数返回后局部引很可能马上就会被释放掉，静态变量中存储的就是一个被释放后的内存地址，成了一个野针对，下次再使用的时候就会造成非法地址的访问，使程序崩溃。请看下面一个例子，错误的缓存了 String 的 Class 引用。

```c++
/*错误的局部引用*/
JNIEXPORT jstring JNICALL Java_com_study_jnilearn_AccessCache_newString
(JNIEnv *env, jobject obj, jcharArray j_char_arr, jint len)
{
    jcharArray elemArray;
    jchar *chars = NULL;
    jstring j_str = NULL;
    static jclass cls_string = NULL;
    static jmethodID cid_string = NULL;
    // 注意：错误的引用缓存
    if (cls_string == NULL) {
        cls_string = (*env)->FindClass(env, "java/lang/String");
        if (cls_string == NULL) {
            return NULL;
        }
    }
    // 缓存String的构造方法ID
    if (cid_string == NULL) {
        cid_string = (*env)->GetMethodID(env, cls_string, "<init>", "([C)V");
        if (cid_string == NULL) {
            return NULL;
        }
    }

   //省略额外的代码.......
    elemArray = (*env)->NewCharArray(env, len);
    // ....
    j_str = (*env)->NewObject(env, cls_string, cid_string, elemArray);
    // 释放局部引用
    (*env)->DeleteLocalRef(env, elemArray);
    return j_str;
}
```

　　上面代码中，我们省略了和我们讨论无关的代码。因为 FindClass 返回一个对 java.lang.String 对象的局部引用，上面代码中缓存 cls_string 做法是错误的。假设一个本地方法 C.f 调用了 newString。

```c++
JNIEXPORT jstring JNICALL
 Java_C_f(JNIEnv *env, jobject this)
 {
     char *c_str = ...;
     ...
     return newString(c_str);
}
```

　　Java_com_study_jnilearn_AccessCache_newString 下面简称 newString。    
C.f 方法返回后，JVM 会释放在这个方法执行期间创建的所有局部引用，也包含对 String 的 Class 引用cls_string。当再次调用 newString 时，newString 所指向引用的内存空间已经被释放，成为了一个野指针，再访问这个指针的引用时，会导致因非法的内存访问造成程序崩溃。

```c++
...
... = C.f(); // 第一次调是OK的
... = C.f(); // 第二次调用时，访问的是一个无效的引用.
...
```

####释放局部引用
　　释放一个局部引用有两种方式，一个是本地方法执行完毕后 JVM 自动释放，另外一个是自己调用 DeleteLocalRef 手动释放。既然 JVM 会在函数返回后会自动释放所有局部引用，为什么还需要手动释放呢？大部分情况下，我们在实现一个本地方法时不必担心局部引用的释放问题，函数被调用完成后，JVM 会自动释放函数中创建的所有局部引用。尽管如此，以下几种情况下，为了避免内存溢出，我们应该手动释放局部引用。

　　JNI 会将创建的局部引用都存储在一个局部引用表中，如果这个表超过了最大容量限制，就会造成局部引用表溢出，使程序崩溃。经测试，Android 上的 JNI 局部引用表最大数量是 512 个。当我们在实现一个本地方法时，可能需要创建大量的局部引用，如果没有及时释放，就有可能导致 JNI 局部引用表的溢出，所以，在不需要局部引用时就立即调用 DeleteLocalRef 手动删除。比如，在下面的代码中，本地代码遍历一个特别大的字符串数组，每遍历一个元素，都会创建一个局部引用，当对使用完这个元素的局部引用时，就应该马上手动释放它。

```c++
for (i = 0; i < len; i++) {
     jstring jstr = (*env)->GetObjectArrayElement(env, arr, i);
     ... /* 使用jstr */
     (*env)->DeleteLocalRef(env, jstr); // 使用完成之后马上释放
}
```

　　在编写 JNI 工具函数时，工具函数在程序当中是公用的，被谁调用你是不知道的。上面 newString 这个函数演示了怎么样在工具函数中使用完局部引用后，调用 DeleteLocalRef 删除。不这样做的话，每次调用 newString 之后，都会遗留两个引用占用空间（elemArray和cls_string，cls_string 不用 static 缓存的情况下）。

　　如果你的本地函数不会返回。比如一个接收消息的函数，里面有一个死循环，用于等待别人发送消息过来**while(true) { if (有新的消息) ｛ 处理之。。。。｝ else { 等待新的消息。。。}}**。如果在消息循环当中创建的引用你不显示删除，很快将会造成 JVM 局部引用表溢出。

　　局部引用会阻止所引用的对象被 GC 回收。比如你写的一个本地函数中刚开始需要访问一个大对象，因此一开始就创建了一个对这个对象的引用，但在函数返回前会有一个大量的非常复杂的计算过程，而在这个计算过程当中是不需要前面创建的那个大对象的引用的。但是，在计算的过程当中，如果这个大对象的引用还没有被释放的话，会阻止 GC 回收这个对象，内存一直占用者，造成资源的浪费。所以这种情况下，在进行复杂计算之前就应该把引用给释放了，以免不必要的资源浪费。

```c++
/* 假如这是一个本地方法实现 */
JNIEXPORT void JNICALL Java_pkg_Cls_func(JNIEnv *env, jobject this)
{
   lref = ...              /* lref引用的是一个大的Java对象 */
   ...                     /* 在这里已经处理完业务逻辑后，这个对象已经使用完了 */
   (*env)->DeleteLocalRef(env, lref); /* 及时删除这个对这个大对象的引用，GC就可以对它回收，并释放相应的资源*/
   lengthyComputation();   /* 在里有个比较耗时的计算过程 */
   return;                 /* 计算完成之后，函数返回之前所有引用都已经释放 */
}
```

####管理局部引用

　　JNI 提供了一系列函数来管理局部引用的生命周期。这些函数包括：EnsureLocalCapacity、NewLocalRef、PushLocalFrame、PopLocalFrame、DeleteLocalRef。JNI 规范指出，任何实现 JNI 规范的 JVM，必须确保每个本地函数至少可以创建 16 个局部引用（可以理解为虚拟机默认支持创建 16 个局部引用）。实际经验表明，这个数量已经满足大多数不需要和 JVM 中内部对象有太多交互的本地方函数。如果需要创建更多的引用，可以通过调用 EnsureLocalCapacity 函数，确保在当前线程中创建指定数量的局部引用，如果创建成功则返回 0，否则创建失败，并抛出 OutOfMemoryError 异常。EnsureLocalCapacity 这个函数是 1.2 以上版本才提供的，为了向下兼容，在编译的时候，如果申请创建的局部引用超过了本地引用的最大容量，在运行时 JVM 会调用 FatalError 函数使程序强制退出。在开发过程当中，可以为 JVM 添加-verbose:jni参数，在编译的时如果发现本地代码在试图申请过多的引用时，会打印警告信息提示我们要注意。在下面的代码中，遍历数组时会获取每个元素的引用，使用完了之后不手动删除，不考虑内存因素的情况下，它可以为这种创建大量的局部引用提供足够的空间。由于没有及时删除局部引用，因此在函数执行期间，会消耗更多的内存。

```c++
/*处理函数逻辑时，确保函数能创建len个局部引用*/
if((*env)->EnsureLocalCapacity(env,len) != 0) {
    ... /*申请len个局部引用的内存空间失败 OutOfMemoryError*/
    return;
}
for(i=0; i < len; i++) {
    jstring jstr = (*env)->GetObjectArrayElement(env, arr, i);
    // ... 使用jstr字符串
    /*这里没有删除在for中临时创建的局部引用*/
}
```

　　另外，除了 EnsureLocalCapacity 函数可以扩充指定容量的局部引用数量外，我们也可以利用 Push/PopLocalFrame 函数对创建作用范围层层嵌套的局部引用。例如，我们把上面那段处理字符串数组的代码用 Push/PopLocalFrame 函数对重写。

```c++
#define N_REFS ... /*最大局部引用数量*/
for (i = 0; i < len; i++) {
    if ((*env)->PushLocalFrame(env, N_REFS) != 0) {
        ... /*内存溢出*/
    }
     jstring jstr = (*env)->GetObjectArrayElement(env, arr, i);
     ... /* 使用jstr */
     (*env)->PopLocalFrame(env, NULL);
}
```
　　PushLocalFrame 为当前函数中需要用到的局部引用创建了一个引用堆栈，（如果之前调用 PushLocalFrame 已经创建了 Frame，在当前的本地引用栈中仍然是有效的)每遍历一次调用(*env)->GetObjectArrayElement(env, arr, i);返回一个局部引用时，JVM 会自动将该引用压入当前局部引用栈中。而 PopLocalFrame 负责销毁栈中所有的引用。这样一来，Push/PopLocalFrame 函数对提供了对局部引用生命周期更方便的管理，而不需要时刻关注获取一个引用后，再调用 DeleteLocalRef 来释放引用。在上面的例子中，如果在处理 jstr 的过程当中又创建了局部引用，则 PopLocalFrame 执行时，这些局部引用将全都会被销毁。在调用 PopLocalFrame 销毁当前 frame 中的所有引用前，如果第二个参数 result 不为空，会由 result 生成一个新的局部引用，再把这个新生成的局部引用存储在上一个 frame 中。请看下面的示例。

```c++
// 函数原型
jobject (JNICALL *PopLocalFrame)(JNIEnv *env, jobject result);

jstring other_jstr;
for (i = 0; i < len; i++) {
    if ((*env)->PushLocalFrame(env, N_REFS) != 0) {
        ... /*内存溢出*/
    }
     jstring jstr = (*env)->GetObjectArrayElement(env, arr, i);
     ... /* 使用jstr */
     if (i == 2) {
        other_jstr = jstr;
     }
    other_jstr = (*env)->PopLocalFrame(env, other_jstr);  // 销毁局部引用栈前返回指定的引用
}
```

　　还要注意的一个问题是，局部引用不能跨线程使用，只在创建它的线程有效。不要试图在一个线程中创建局部引用并存储到全局引用中，然后在另外一个线程中使用。

####全局引用

　　全局引用可以跨方法、跨线程使用，直到它被手动释放才会失效。同局部引用一样，也会阻止它所引用的对象被 GC 回收。与局部引用创建方式不同的是，只能通过 NewGlobalRef 函数创建。下面这个版本的 newString 演示怎么样使用一个全局引用。

```c++
JNIEXPORT jstring JNICALL Java_com_study_jnilearn_AccessCache_newString
(JNIEnv *env, jobject obj, jcharArray j_char_arr, jint len)
{
    // ...
    jstring jstr = NULL;
    static jclass cls_string = NULL;
    if (cls_string == NULL) {
        jclass local_cls_string = (*env)->FindClass(env, "java/lang/String");
        if (cls_string == NULL) {
            return NULL;
        }

        // 将java.lang.String类的Class引用缓存到全局引用当中
        cls_string = (*env)->NewGlobalRef(env, local_cls_string);

        // 删除局部引用
        (*env)->DeleteLocalRef(env, local_cls_string);

        // 再次验证全局引用是否创建成功
        if (cls_string == NULL) {
            return NULL;
        }
    }

    // ....
    return jstr;
}
```

####弱全局引用
　　弱全局引用使用 NewGlobalWeakRef 创建，使用 DeleteGlobalWeakRef 释放。下面简称弱引用。与全局引用类似，弱引用可以跨方法、线程使用。但与全局引用很重要不同的一点是，弱引用不会阻止 GC 回收它引用的对象。在newString 这个函数中，我们也可以使用弱引用来存储 String 的 Class 引用，因为 java.lang.String 这个类是系统类，永远不会被 GC 回收。当本地代码中缓存的引用不一定要阻止 GC 回收它所指向的对象时，弱引用就是一个最好的选择。假设，一个本地方法mypkg.MyCls.f需要缓存一个指向类mypkg.MyCls2的引用，如果在弱引用中缓存的话，仍然允许mypkg.MyCls2这个类被 unload，因为弱引用不会阻止 GC 回收所引用的对象。请看下面的代码段。

```c++
JNIEXPORT void JNICALL
Java_mypkg_MyCls_f(JNIEnv *env, jobject self)
{
    static jclass myCls2 = NULL;
    if (myCls2 == NULL)
    {
        jclass myCls2Local = (*env)->FindClass(env, "mypkg/MyCls2");
        if (myCls2Local == NULL)
        {
            return; /* 没有找到mypkg/MyCls2这个类 */
        }
        myCls2 = NewWeakGlobalRef(env, myCls2Local);
        if (myCls2 == NULL)
        {
            return; /* 内存溢出 */
        }
    }
    ... /* 使用myCls2的引用 */
}
```

　　我们假设 MyCls 和 MyCls2 有相同的生命周期（例如，他们可能被相同的类加载器加载），因为弱引用的存在，我们不必担心 MyCls 和它所在的本地代码在被使用时，MyCls2 这个类出现先被 unload，后来又会 preload 的情况。当然，如果真的发生这种情况时（MyCls 和 MyCls2 此时的生命周期不同），我们在使用弱引用时，必须先检查缓存过的弱引用是指向活动的类对象，还是指向一个已经被 GC 给 unload 的类对象。下面马上告诉你怎样检查弱引用是否活动，即引用的比较。

####引用比较
　　给定两个引用（不管是全局、局部还是弱全局引用），我们只需要调用 IsSameObject 来判断它们两个是否指向相同的对象。例如：`（*env)->IsSameObject(env, obj1, obj2)`，如果 obj1 和 obj2 指向相同的对象，则返回 JNI_TRUE（或者 1），否则返回 JNI_FALSE（或者 0）。有一个特殊的引用需要注意：NULL，JNI 中的 NULL 引用指向 JVM 中的 null 对象。如果 obj 是一个局部或全局引用，使用(*env)->IsSameObject(env, obj, NULL) 或者obj == NULL 来判断 obj 是否指向一个 null 对象即可。但需要注意的是，IsSameObject 用于弱全局引用与 NULL 比较时，返回值的意义是不同于局部引用和全局引用的。

```c++
jobject local_obj_ref = (*env)->NewObject(env, xxx_cls,xxx_mid);
jobject g_obj_ref = (*env)->NewWeakGlobalRef(env, local_ref);
// ... 业务逻辑处理
jboolean isEqual = (*env)->IsSameObject(env, g_obj_ref, NULL);
```

　　在上面的 IsSameObject 调用中，如果 g_obj_ref 指向的引用已经被回收，会返回 JNI_TRUE，如果 wobj 仍然指向一个活动对象，会返回 JNI_FALSE。

####释放全局引用
　　每一个 JNI 引用被建立时，除了它所指向的 JVM 中对象的引用需要占用一定的内存空间外，引用本身也会消耗掉一个数量的内存空间。作为一个优秀的程序员，我们应该对程序在一个给定的时间段内使用的引用数量要十分小心。短时间内创建大量而没有被立即回收的引用很可能就会导致内存溢出。     当我们的本地代码不再需要一个全局引用时，应该马上调用 DeleteGlobalRef 来释放它。如果不手动调用这个函数，即使这个对象已经没用了，JVM 也不会回收这个全局引用所指向的对象。      同样，当我们的本地代码不再需要一个弱全局引用时，也应该调用 DeleteWeakGlobalRef 来释放它，如果不手动调用这个函数来释放所指向的对象，JVM 仍会回收弱引用所指向的对象，但弱引用本身在引用表中所占的内存永远也不会被回收。

####管理引用的规则
　　前面对三种引用已做了一个全面的介绍，下面来总结一下引用的管理规则和使用时的一些注意事项，使用好引用的目的就是为了减少内存使用和对象被引用保持而不能释放，造成内存浪费。所以在开发当中要特别小心！

　　通常情况下，有两种本地代码使用引用时要注意：

　　直接实现Java层声明的native函数的本地代码 当编写这类本地代码时，要当心不要造成全局引用和弱引用的累加，因为本地方法执行完毕后，这两种引用不会被自动释放。
被用在任何环境下的工具函数。例如：方法调用、属性访问和异常处理的工具函数等。
编写工具函数的本地代码时，要当心不要在函数的调用轨迹上遗漏任何的局部引用，因为工具函数被调用的场合和次数是不确定的，一量被大量调用，就很有可能造成内存溢出。所以在编写工具函数时，请遵守下面的规则：

　　一个返回值为基本类型的工具函数被调用时，它决不能造成局部、全局、弱全局引用被回收的累加。   
　　当一个返回值为引用类型的工具函数被调用时，它除了返回的引用以外，它决不能造成其它局部、全局、弱引用的累加。    
　　对于工具函数来说，为了使用缓存技术而创建一些全局引用或者弱全局引用是正常的。如果一个工具函数返回的是一个引用，我们应该写好注释详细说明返回引用的类型，以便于使用者更好的管理它们。下面的代码中，频繁地调用工具函数 GetInfoString，我们需要知道 GetInfoString 返回引用的类型是什么，以便于每次使用完成后调用相应的 JNI 函数来释放掉它。

```c++
 while (JNI_TRUE) {
     jstring infoString = GetInfoString(info);
     ... /* 处理infoString */
     ??? /* 使用完成之后，调用DeleteLocalRef、DeleteGlobalRef、DeleteWeakGlobalRef哪一个函数来释放这个引用呢？*/
}
```

　　函数 NewLocalRef 有时被用来确保一个工具函数返回一个局部引用。我们改造一下 newString 这个函数，演示一下这个函数的用法。下面的 newString 是把一个被频繁调用的字符串“CommonString”缓存在了全局引用里。

```c++
JNIEXPORT jstring JNICALL Java_com_study_jnilearn_AccessCache_newString
{
    static jstring result;
    /* 使用wstrncmp函数比较两个Unicode字符串 */
    if (wstrncmp("CommonString", chars, len) == 0)
    {
        /* 将"CommonString"这个字符串缓存到全局引用中 */
        static jstring cachedString = NULL;
        if (cachedString == NULL)
        {
            /* 先创建"CommonString"这个字符串 */
            jstring cachedStringLocal = ...;
            /* 然后将这个字符串缓存到全局引用中 */
            cachedString = (*env)->NewGlobalRef(env, cachedStringLocal);
        }
        // 基于全局引用创建一个局引用返回，也同样会阻止GC回收所引用的这个对象，因为它们指向的是同一个对象
        return (*env)->NewLocalRef(env, cachedString);  
    }
    ... 
    return result;
}
```

　　在管理局部引用的生命周期中，Push/PopLocalFrame 是非常方便且安全的。我们可以在本地函数的入口处调用PushLocalFrame，然后在出口处调用 PopLocalFrame，这样的话，在函数内任何位置创建的局部引用都会被释放。而且，这两个函数是非常高效的，强烈建议使用它们。需要注意的是，如果在函数的入口处调用了PushLocalFrame，记住要在函数所有出口（有 return 语句出现的地方）都要调用 PopLocalFrame。在下面的代码中，对 PushLocalFrame 的调用只有一次，但调用 PopLocalFrame 确有多次，当然你也可以使用 goto 语句来统一处理。

```c++
jobject f(JNIEnv *env, ...)
{
    jobject result;
    if ((*env)->PushLocalFrame(env, 10) < 0)
    {
        /* 调用PushLocalFrame获取10个局部引用失败，不需要调用PopLocalFrame */
        return NULL;
    }
    ...
    result = ...; // 创建局部引用result
    if (...)
    {
        /* 返回前先弹出栈顶的frame */
        result = (*env)->PopLocalFrame(env, result);
        return result;
    }
    ...
    result = (*env)->PopLocalFrame(env, result);
    /* 正常返回 */
    return result;
}
```
　　上面的代码同样演示了函数 PopLocalFrame 的第二个参数的用法，局部引用 result 一开始在 PushLocalFrame 创建在当前 frame 里面，而把 result 传入 PopLocalFrame 中时，PopLocalFrame 在弹出当前的 frame 前，会由 result 生成一个新的局部引用，再将这个新生成的局部引用存储在上一个 frame 当中。