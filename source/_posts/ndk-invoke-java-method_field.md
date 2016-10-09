title: "NDK开发 - C/C++ 访问 Java 变量和方法"
date: 2016-04-08 10:49:56
categories: android
tags: [android, ndk, jni]
toc: true
description: 上一篇有提到 JNI 访问引用数组，涉及了 C/C++ 访问 Java 实例的方法和变量。虽然在之前的开发中，并没有用到 C/C++ 范围 Java 层数据，但是这部分内容还是很有用的。 

---

　　C/C++ 访问 Java 层的方法和变量主要分为实例和静态。调用静态变量和方法，直接通过 `类名.方法/类名.变量`，而调用实例变量和方法，需要先实例化对象才可以调用。另外对于继承的子类访问父类的方法也是可以通过 JNI 实现，虽然可能用的不多。

###0x00 调用静态变量和方法
　　静态数据的访问其实比较简单。当我们在运行一个 Java 程序时，JVM 会先将程序运行时所要用到所有相关的 class 文件加载到 JVM 中，并采用按需加载的方式加载，也就是说某个类只有在被用到的时候才会被加载，这样设计的目的也是为了提高程序的性能和节约内存。本地代码也是按照上面的流程来访问类的静态方法或实例方法的。下面直接上代码：   
####Java层代码

```java
 //调用静态方法/变量
 NativeMethod.invokeStaticFieldAndMethod(23, "gnaix");
 Log.d(TAG, "field value:" + Util.STATIC_FIELD);
```

```java
/**
 * 名称: Util
 * 描述:
 *
 * @author xiangqing.xue
 * @date 16/3/11
 */
public class Util {
    private static String TAG = "Util";

    public static int STATIC_FIELD = 300;

    public static String getStaticMethod(String name, int age){
        String str =  "Hello "+ name + ", age:" + age;
        Log.d(TAG, str);
        return str;
    }
}
```
####Native代码

```c++
JNIEXPORT void JNICALL Java_com_example_gnaix_ndk_NativeMethod_invokeStaticFieldAndMethod
        (JNIEnv *env, jclass object, jint age, jstring name){

    jclass clazz = NULL;
    jfieldID fid = NULL;
    jmethodID mid = NULL;
    jstring j_result = NULL;

    //1. 获取Util类的Class引用
    clazz = env->FindClass("com/example/gnaix/ndk/Util");
    if(clazz == NULL){
        LOGD("clazz null");
        return;
    }

    //2. 获取静态变量属性ID和方法ID
    fid = env->GetStaticFieldID(clazz, "STATIC_FIELD", "I");
    if(fid == NULL){
        LOGD("fieldID null");
        return;
    }
    mid = env->GetStaticMethodID(clazz, "getStaticMethod", "(Ljava/lang/String;I)Ljava/lang/String;");
    if(mid == NULL){
        LOGD("method null");
        return;
    }

    //3. 获取静态变量值和调用静态方法
    jint num = env->GetStaticIntField(clazz, fid);
    LOGD("static field value: %d", num);
    j_result = (jstring)env->CallStaticObjectMethod(clazz, mid, name, age);
    const char *buf_result = env->GetStringUTFChars(j_result, 0);
    LOGD("static: %s", buf_result);
    env->ReleaseStringUTFChars(j_result, buf_result);

    //4. 修改静态变量值
    env->SetStaticIntField(clazz, fid, 100);

    //5. 释放局部引用
    env->DeleteLocalRef(clazz);
    env->DeleteLocalRef(j_result);
}
```

####运行结果
<img width=700px height=100px src="https://gnaix92.github.io/blog_images/ndk/9.png" style="display:inline-block"/>

####代码解析
第一步：调用 `jclass FindClass(const char* name)` 参数为 Java 类的 classPath 只是把`.`换成`/`，获取jclass。  
 
第二步：获取变量和方法的签名ID   
`jfieldID GetStaticFieldID(jclass clazz, const char* name, const char* sig)`     
` jmethodID GetStaticMethodID(jclass clazz, const char* name, const char* sig)`    
参数说明：

- clazz: Java类的 Class 对象（第一步获取的jclass对象）
- name: 变量名/方法名
- sig: 变量名/方法名签名ID   

签名获取的方法在上一篇有提到过:[NDK开发 - JNI数组数据处理](https://gnaix92.github.io/2016/04/07/NDK%E5%BC%80%E5%8F%91-JNI%E6%95%B0%E7%BB%84%E6%95%B0%E6%8D%AE%E5%A4%84%E7%90%86/)

第三步1：获取静态变量的值
`jint GetStaticIntField(jclass clazz, jfieldID fieldID)`方法获取int类型静态变量的值，相同的函数还有 `GetStaticLongField`, `GetStaticFloatField`, `GetStaticDoubleField`等，引用类型都是用 `GetStaticObjectField`。   

第三步2：调用静态函数，通过 JNI 的 `CallStaticObjectMethod(clazz, mid, name, age)` 函数调用 Java 的 `getStaticMethod` 静态函数，第一个参数为 Java 的 Class 对象，mid 为方法的签名， 后面的参数为 `getStaticMethod` 的参数，返回值为 jstring。同样也有 `CallStaticXXMethod` 方法调用表示不同的返回值。

第四步：通过 `SetStaticIntField`改变静态变量的值。

第五步：释放局部变量

###调用实例变量和方法
　　调用实例变量与方法与范围静态的类型，不过多了一个实例对象的过程。  
  
####Java 层代码
```java
 //调用实例方法/变量
 NativeMethod.invokeJobject("gnaix", 23);
```
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

####Native代码

```c++
JNIEXPORT void JNICALL Java_com_example_gnaix_ndk_NativeMethod_invokeJobject
        (JNIEnv *env, jclass object, jstring name, jint age){
    jclass clazz = NULL;
    jobject jobj = NULL;
    jfieldID fid_name = NULL;
    jfieldID fid_age = NULL;
    jmethodID mid_construct = NULL;
    jmethodID mid_to_string = NULL;
    jstring j_new_str;
    jstring j_result;

    //1. 获取Person类Class引用
    clazz = env->FindClass("com/example/gnaix/ndk/Person");
    if(clazz == NULL){
        LOGD("clazz null");
        return;
    }

    //2. 获取类的默认构造函数ID
    mid_construct = env->GetMethodID(clazz, "<init>", "()V");
    if(mid_construct == NULL){
        LOGD("construct null");
        return;
    }

    //3. 获取实例方法ID和变量ID
    fid_name = env->GetFieldID(clazz, "name", "Ljava/lang/String;");
    fid_age = env->GetFieldID(clazz, "age", "I");
    mid_to_string = env->GetMethodID(clazz, "toString", "()Ljava/lang/String;");
    if(fid_age==NULL || fid_name==NULL || mid_to_string==NULL){
        LOGD("age|name|toString null");
        return;
    }

    //4. 创建该类的实例
    jobj = env->NewObject(clazz, mid_construct);
    if(jobj == NULL){
        LOGD("jobject null");
        return;
    }

    //5. 修改变量值
    env->SetIntField(jobj, fid_age, 23);
    j_new_str = env->NewStringUTF("gnaix");
    env->SetObjectField(jobj, fid_name, j_new_str);

    //6. 调用实例方法
    j_result = (jstring)env->CallObjectMethod(jobj, mid_to_string);
    const char *buf_result = env->GetStringUTFChars(j_result, 0);
    LOGD("object: %s", buf_result);
    env->ReleaseStringUTFChars(j_result, buf_result);

    //7. 释放局部引用
    env->DeleteLocalRef(clazz);
    env->DeleteLocalRef(jobj);
    env->DeleteLocalRef(j_new_str);
    env->DeleteLocalRef(j_result);
}
```

####运行结果
<img width=700px height=100px src="https://gnaix92.github.io/blog_images/ndk/10.png" style="display:inline-block"/>

####代码解析
　　访问实例变量和方法没有很大区别多了一个实例化的步骤。每个类会生成一个默认的构造函数，调用的时候用 `<init>` ，签名ID是 `()V`。如果是自定义的构造函数需要用相应的签名ID。

```c++
//2. 获取类的默认构造函数ID
mid_construct = env->GetMethodID(clazz, "<init>", "()V");
if(mid_construct == NULL){
    LOGD("construct null");
    return;
}

//4. 创建该类的实例
jobj = env->NewObject(clazz, mid_construct);
if(jobj == NULL){
    LOGD("jobject null");
    return;
}
```
###0x01 调用构造方法和父类实例方法

####Java 代码
　　定义了 Animal 和 Dog 两个类，Animal 定义了一个 String 形参的构造方法，一个成员变量 name、两个成员函数 run 和 getName，Dog 继承自 Animal，并重写了 run 方法。在 JNI 中实现创建 Dog 对象的实例，调用 Animal 类的 run 和 getName 方法。代码如下所示:

```java
/**
 * 名称: Animal
 * 描述:
 *
 * @author xiangqing.xue
 * @date 16/4/1
 */
public class Animal {
    private String TAG = "Animal";

    protected String name;

    public Animal(String name){
        this.name = name;
        Log.d(TAG, "Animal Construct call...");
    }

    public String getName(){
        Log.d(TAG, "Animal.getName Call...");
        return this.name;
    }

    public void run(){
        Log.d(TAG, "Animal.run...");
    }
}
```

```java
/**
 * 名称: Dog
 * 描述:
 *
 * @author xiangqing.xue
 * @date 16/4/1
 */
public class Dog extends Animal{

    private String TAG = "Dog";

    public Dog(String name) {
        super(name);
        Log.d(TAG, "Dog Construct call...");
    }

    @Override
    public String getName() {
        return "My name is " + this.name;
    }

    @Override
    public void run() {
        Log.d(TAG, name + "Dog.run...");
    }
}
```
####Native 代码

```c++
JNIEXPORT void JNICALL Java_com_example_gnaix_ndk_NativeMethod_callSuperInstanceMethod
        (JNIEnv *env, jclass jobj){
    jclass cls_dog;
    jclass cls_animal;
    jmethodID mid_dog_init;
    jmethodID mid_run;
    jmethodID mid_get_name;
    jstring c_str_name;
    jobject obj_dog;
    const char *name = NULL;

    //1. 获取Dog类Class引用
    cls_dog = env->FindClass("com/example/gnaix/ndk/Dog");
    if(cls_dog == NULL){
        LOGD("cls_dog null");
        return;
    }

    //2. 获取Dog类构造方法ID
    mid_dog_init = env->GetMethodID(cls_dog, "<init>", "(Ljava/lang/String;)V");
    if(mid_dog_init == NULL){
        LOGD("cls_dog init null");
        return;
    }

    //3. 创建Dog对象实例
    c_str_name = env->NewStringUTF("Tom");
    obj_dog = env->NewObject(cls_dog, mid_dog_init, c_str_name);
    if(obj_dog == NULL){
        LOGD("obj_dog null");
        return;
    }

    //4. 获取animal类Class引用
    cls_animal = env->FindClass("com/example/gnaix/ndk/Animal");
    if(cls_animal == NULL){
        LOGD("cls_dog null");
        return;
    }

    //5. 获取父类run方法ID并调用
    mid_run = env->GetMethodID(cls_animal, "run", "()V");
    if(mid_run == NULL){
        LOGD("mid_run null");
        return;
    }
    env->CallNonvirtualVoidMethod(obj_dog, cls_animal, mid_run);

    //6. 调用父类getName方法
    mid_get_name = env->GetMethodID(cls_animal, "getName", "()Ljava/lang/String;");
    if(mid_get_name == NULL){
        LOGD("mid_get_name null");
        return;
    }
    c_str_name = (jstring) env->CallNonvirtualObjectMethod(obj_dog, cls_animal, mid_get_name);
    name = env->GetStringUTFChars(c_str_name, 0);
    LOGD("In C: Animal Name is %s", name);

    //7. 释放从java层获取到的字符串所分配的内存
    env->ReleaseStringUTFChars(c_str_name, name);

    //8. 释放局部变量
    env->DeleteLocalRef(cls_animal);
    env->DeleteLocalRef(cls_dog);
    env->DeleteLocalRef(c_str_name);
    env->DeleteLocalRef(obj_dog);
}
```

####运行结果
<img width=700px height=100px src="https://gnaix92.github.io/blog_images/ndk/11.png" style="display:inline-block"/>

####代码解析

#####调用构造方法

调用构造方法和调用对象的实例方法方式是相似的，传入”< init >”作为方法名查找类的构造方法ID，然后调用JNI函数NewObject调用对象的构造函数初始化对象。如下代码所示：

```c++
 //2. 获取Dog类构造方法ID
mid_dog_init = env->GetMethodID(cls_dog, "<init>", "(Ljava/lang/String;)V");
if(mid_dog_init == NULL){
    LOGD("cls_dog init null");
    return;
}

//3. 创建Dog对象实例
c_str_name = env->NewStringUTF("Tom");
obj_dog = env->NewObject(cls_dog, mid_dog_init, c_str_name);
if(obj_dog == NULL){
    LOGD("obj_dog null");
    return;
}
```

#####调用父类实例方法
　　如果一个方法被定义在父类中，在子类中被覆盖，也可以调用父类中的这个实例方法。JNI 提供了一系列函数CallNonvirtualXXXMethod 来支持调用各种返回值类型的实例方法。调用一个定义在父类中的实例方法，须遵循下面的步骤。    
　　使用 GetMethodID 函数从一个指向父类的 Class 引用当中获取方法 ID。

```c++
//4. 获取animal类Class引用
cls_animal = env->FindClass("com/example/gnaix/ndk/Animal");
if(cls_animal == NULL){
    LOGD("cls_dog null");
    return;
}

//5. 获取父类run方法ID并调用
mid_run = env->GetMethodID(cls_animal, "run", "()V");
if(mid_run == NULL){
    LOGD("mid_run null");
    return;
}
```
　　传入子类对象、父类 Class 引用、父类方法 ID 和参数，并调用 `CallNonvirtualVoidMethod`、 `CallNonvirtualBooleanMethod`、`CallNonvirtualIntMethod` 等一系列函数中的一个。其中`CallNonvirtualVoidMethod` 也可以被用来调用父类的构造函数。

```c++
env->CallNonvirtualVoidMethod(obj_dog, cls_animal, mid_run);
```