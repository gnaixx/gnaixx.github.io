title: "NDK开发 - JNI数组数据处理"
date: 2016-04-07 14:46:34
categories: android
tags: [android, ndk, jni]
toc: true
description: 很多时候利用 NDK 开发都是为了对数据进行加密操作，因为单纯的 Java 太容易被反编译了，加密算法也就很容易被破解，而利用 C/C++ 开发可以加大破解难度。文件的数据加密就需要通过 byte 数组传给 JNI。

---
　　JNI 中的数组分为基本类型数组和对象数组，它们的处理方式是不一样的，基本类型数组中的所有元素都是 JNI 的基本数据类型，可以直接访问。而对象数组中的所有元素是一个类的实例或其它数组的引用，和字符串操作一样，不能直接访问 Java 传递给 JNI 层的数组，必须选择合适的 JNI 函数来访问和设置 Java 层的数组对象。

###访问基本类型数组
Java 代码：

```java
//调用数组
byte array[] = {'A', 'B', 'C', 'D', 'E'};
byte[] resutl = NativeMethod.getByteArray(array);
for (int i = 0; i < array.length; i++) {
    Log.d(TAG, "ARRAY : " + array[i] + "->" + resutl[i]);
} 
```

native 代码：

```c++
JNIEXPORT jbyteArray JNICALL Java_com_example_gnaix_ndk_NativeMethod_getByteArray
        (JNIEnv *env, jclass object, jbyteArray j_array){
    //1. 获取数组指针和长度
    jbyte *c_array = env->GetByteArrayElements(j_array, 0);
    int len_arr = env->GetArrayLength(j_array);

    //2. 具体处理
    jbyteArray c_result = env->NewByteArray(len_arr);
    jbyte buf[len_arr];
    for(int i=0; i<len_arr; i++){
        buf[i] = c_array[i] + 1;
    }

    //3. 释放内存
    env->ReleaseByteArrayElements(j_array, c_array, 0);

    //4. 赋值
    env->SetByteArrayRegion(c_result, 0, len_arr, buf);
    return c_result;
}
```
运行结果：
<img width=700px height=100px src="http://gnaix92.github.io/blog_images/ndk/5.png" style="display:inline-block"/>

　　示例中，从 Java 层中传进去了一个数组，参数类型是 byte[], 对应 JNI 中 jbyteArray 类型。利用 `GetByteArrayElements` 函数获取数组指针，第二个参数返回的数组指针是原始数组，还是拷贝原始数据到临时缓冲区的指针，如果是 JNI_TRUE：表示临时缓冲区数组指针，JNI_FALSE：表示临时原始数组指针。开发当中，我们并不关心它从哪里返回的数组指针，这个参数填 NULL 即可，但在获取到的指针必须做校验。
类似的函数还有 `GetIntArrayElements` , `GetFloatArrayElements` , `GetDoubleArrayElements` 等。   
　　然后对数据进行了处理，并且释放了数组内存。`ReleaseByteArrayElements` 是与 `GetByteArrayElements` 对应使用的。

　　除了上述方法还可以用 JNI 中的 `GetXXArrayRegion` 函数进行实现数据操作, 对应的 `SetXXArrayRegion` 上述也有提到过。

```c++
JNIEXPORT jint JNICALL Java_com_example_gnaix_ndk_NativeMethod_getByteArray
        (JNIEnv *env, jclass object, jbyteArray j_array){
	jbyte *c_array;
    jint len_arr;

    //1. 获取数组长度
    len_arr = env->GetArrayLength(j_array);

    //2. 申请缓冲区
    c_array = (jbyte *) malloc(sizeof(jbyte) * len_arr);

    //3. 初始化缓冲区
    memset(c_array, 0, sizeof(jbyte)*len_arr);

    //4. 拷贝java数组数据
    env->GetByteArrayRegion(j_array, 0, len_arr, c_array);

    //5. 计算总和
    int sum = 0;
    for(int i=0; i<len_arr; i++){
        sum += c_array[i];
    }

    //6. 释放缓冲区
    free(c_array);
    return sum;
}
```

###访问引用类型数组
　　JNI 提供了两个函数来访问对象数组，GetObjectArrayElement 返回数组中指定位置的元素，SetObjectArrayElement 修改数组中指定位置的元素。与基本类型不同的是，我们不能一次得到数据中的所有对象元素或者一次复制多个对象元素到缓冲区。因为字符串和数组都是引用类型，只能通过 Get/SetObjectArrayElement 这样的 JNI 函数来访问字符串数组或者数组中的数组元素。    
　　下面的例子同调用 Native 方法 创建一个对象数组，返回后打印这个数组内容：

```java
//返回对象数组
Person persons[] =  NativeMethod.getPersons();
for(Person person : persons){
    person.toString();
}
```

Person 类：

```java
/**
 * 名称: Person
 * 描述:
 *
 * @author xiangqing.xue
 * @date 16/3/10
 */
public class Person {
    private String TAG = "Person";

    public String name;
    public int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String toString(){
        String result =  "Name: " + name + ", age: " + age;
        Log.d(TAG, result);
        return result;
    }
}
```
Native 方法：

```c++
JNIEXPORT jobjectArray JNICALL Java_com_example_gnaix_ndk_NativeMethod_getPersons
        (JNIEnv *env, jclass object){
    jclass clazz = NULL;
    jobject jobj = NULL;
    jmethodID mid_construct = NULL;
    jfieldID fid_age = NULL;
    jfieldID fid_name = NULL;
    jstring j_name;

    //1. 获取Person类的Class引用
    clazz = env->FindClass("com/example/gnaix/ndk/Person");
    if(clazz == NULL){
        LOGD("clazz null");
        return NULL;
    }

    //2. 获取类的默认构造函数ID
    mid_construct = env->GetMethodID(clazz, "<init>", "()V");
    if(mid_construct == NULL){
        LOGD("construct null");
        return NULL;
    }

    //3. 获取实例方法ID和变量ID
    fid_name = env->GetFieldID(clazz, "name", "Ljava/lang/String;");
    fid_age = env->GetFieldID(clazz, "age", "I");
    if(fid_age==NULL || fid_name==NULL){
        LOGD("age|name null");
        return NULL;
    }

    //4. 处理单个对象并添加到数组
    int size = 5;
    jobjectArray obj_array = env->NewObjectArray(size, clazz, 0);
    for(int i=0; i<size; i++){
        jobj = env->NewObject(clazz, mid_construct);
        if(jobj == NULL){
            LOGD("jobject null");
            return NULL;
        }
        env->SetIntField(jobj, fid_age, 23 + i);
        j_name = env->NewStringUTF("gnaix");
        env->SetObjectField(jobj, fid_name, j_name);
        env->SetObjectArrayElement(obj_array, i, jobj);
    }


    //5. 释放局部变量
    env->DeleteLocalRef(clazz);
    env->DeleteLocalRef(jobj);
    return obj_array;
}
```

运行结果：
<img width=700px height=100px src="http://gnaix92.github.io/blog_images/ndk/6.png" style="display:inline-block"/>

　　Native 方法首先调用 JNI 函数 `FindClass` 获取 Person 类的引用，如果 Person 类加载失败的话， FindClass 会返回 NULL, 然后抛出一个 java.lang.NoClassDefFoundError 异常。    
　　接下来通过 `GetMethodID` 获取了类的默认构造函数的ID（下一节会介绍）。并且通过 `GetFieldId` 获取了Person变量的ID，用于后面的赋值。    
　　调用 `NewObjectArray` 创建一个数组，在for循环中 `NewObject` 实例化 Person类，并通过 `SetXXField` 函数给实例变量赋值。`SetObjectArrayElement` 将实例化对象插入数组。   
　　最后调用 `DeleteLocalRef` 方法释放局部变量。 `DeleteLocalRef` 将新创建的 引用从引用表中移除。在 JNI 中，只有 jobject 以及子类属于引用变量，会占用引用表的空间，jint，jfloat，jboolean 等都是基本类型变量，不会占用引用表空间，即不需要释放。引用表最大空间为 512 个，如果超出这个范围，JVM 就会挂掉。

###方法签名
　　在上面的的例子中，在调用实例变量或者方法，都必须传入一个 jmethodID 的参数。因为在 Java 中存在方法重载（方法名相同，参数列表不同），所以要明确告诉 JVM 调用的是类或实例中的哪一个方法。调用 JNI 的 GetMethodID 函数获取一个 jmethodID 时，需要传入一个方法名称和方法签名，方法名称就是在 Java 中定义的方法名，方法签名的格式为：(形参参数类型列表)返回值。形参参数列表中，引用类型以 L 开头，后面紧跟类的全路径名（需将.全部替换成/），<font color=red>以分号结尾</font>。  
　　可以通过 `javap` 命令获取类的签名，以 Person 为例：

```shell
 javap -s -p app.build.intermediates.classes.all.debug.com.example.gnaix.ndk.Person
```
参数说明:    

- -s: 输出内部类型签名  
- -p: 显示所有类和成员

<img width=500px height=700px src="http://gnaix92.github.io/blog_images/ndk/7.png" style="display:inline-block"/> 

Java 基本类型与方法签名中参数类型和返回值类型的映射关系如下：

| Field Descriptor | Java Language Type |
| ---------------- | ------------------ |
| Z | boolean |
| B | byte |
| C | char |
| S | short |
| I | int |
| J | long |
| F | float |
| D | double |

　　比如，`String fun(int a, float b, boolean c, String d)` 对应的 JNI 方法签名为：`(IFZLjava/lang/String;)Ljava/lang/String;`。