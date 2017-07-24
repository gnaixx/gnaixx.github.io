---
title: Android.mk 配置参数
tags: [android, ndk, makefile]
toc: true
date: 2016-11-06 21:05:48
categories: android
description: Android.mk 的可配置参数会比较多，但是常用的可能很少。在进行多项目 ndk 共享的时候，如果对各个参数比较了解，对项目的结构优化有不小的好处。

---

## 0x00 LOCAL_PATH
根据 Android 构建系统要求，Android.mk 文档必须以 LOCAL_PATH 变量的定义开头。      

```xml
LOCAL_PATH := $(call my-dir)
```
Android 构建系统利用 LOCAL_PATH 来定位源文件。因为将改变量设置为硬编码值不合适，所以 Android 构建系统提供了一个名为 my-dir 的宏功能。通过将该变量设置为 my-dir 宏功能的返回值，可以将其放在当前目录下。

## 0x01 CLEAR_VARS
Android 构建系统将 CLEAR_VARS 变量设置为 clear-vars.mk 片段的位置。包含 Makefile 片段可以清除除了 LOCAL_PATH 以外的LOCAL_name 变量，例如 LOCAL_MODULE 与 LOCAL_SRC_FILE 等。     

```xml
Include $(CLEAR_VARS)
```
这样做是因为 Android 构建系统可以在单次执行中解析多个构建文件和模式定义，LOCAL_<name> 是全局变量。清除它们可以避免冲突，每一个原生组件被称为一个模块。

## 0x02 LOCAL_MODULE
改变了是用来给这些模块设定一个唯一的名称。下面的代码将该模块的名称设为 hello-jni:
    
```xml
LOCAL_MODULE := hello-jni
```
因为模块名称也被用于给构建过程的所生成的文件命名，所以构建系统给文件添加适当的前缀和后缀。本例中， hello-jni 模块会生成一个共享文件且构建系统会将它命名为 libhello-jni.so。

## 0x03 LOCAL_SRC_FILES
该变量用来建立和组装这个模块的源文件列表。     

```xml
LOCAL_SRC_FILES := hello-jni.c
```
这里，hello-jni 模块只有一个源文件生成，而 LOCAL_SRC_FILES 变量可以包含用空格分开的多个源文件名。

## 0x04 BUILD_SHARED_LIBRARY
为了建立可供主应用程序使用的模块，必须将该模块变成共享库。Android NDK 构建系统将 BUILD_SHARED_LIBRARY 变量设置成 build_shared_libray.mk 文件保存的位置。改 Makefile 片段包含了将源文件构建和组装成共享库的必要过程。     

```xml
include $(BUILD_SHARED_LIBRARY)
```

## 0x05 BUILD_STATIC_LIBRARY
建立静态库，Android 应用并不直接使用静态库，一般是将第三方代码作为静态库引入动态库中使用。

```xml
include $(BUILD_STATIC_LIBRARY)
```

## 0x06 引入共享库和静态库

```xml
LOCAL_SHARED_LIBRARY := avilib_share
LOCAL_STATIC_LIBRARY := avilib_static
```

## 0x07 多项目共享模块
多个项目共享某个模块时，先将源码导入`%NDK_HOME%/source`,配置好对应的 Android.mk 文件。在需要的项目中添加：

```xml
LOCAL_SHARED_LIBRARIES := avilib
$(call import-module, transcode/avilib) 
```

## 0x08 用 Prebuilt 库
将已编译好的 so 文件作为项目的共享模块：

```xml
LOCAL_PATH := $(call my-dir)
#第三方模块
include $(CLEAR_VARS)
LOCAL_MODULE := avilib
LOCAL_SRC_FILES := libavilib.so

include $(RREBUILT_SHARED_LIBRARY)
```

## 0x09 构建可执行文件
```xml
include $(BUILD_EXECUTABLE)
```

## 0x10 其他构建系统变量
```xml
#目标 CPU 体系结构名称
TARGET_ARCH := arm

#目标 Android 平台名称
TARGET_PLARFORM := andoid-3

#目标 CPU 体系结构和 ABI 名称
TARGET_ARCH_ABI := armeabi-v7a

#目标平台和 ABI 串联
TARGET_ABI := android-3-armeabi-v7a

#重定向输出文件名（默认为 LOCAL_MODULE）
LOCAL_MODULE_FILENAME := avilib

#指定 C++ 源码文件扩展名
LOCAL_CPP_EXTENSION := .cpp .cxx

#指定模块所依赖的具体 C++ 特性
LOCAL_CPP_FEATURES := rtti

#可选目录列表，NDK安装目录的相对路径，用来搜索头文件
LOCAL_C_INCLUDES := sources/shared-module

#添加编译器标志（添加宏定义）
LOCAL_CFLAGS := -DNDEBUG -DPROT=1234

#添加编译器标志（添加宏定义）仅C++
LOCAL_CPP_CFLAGS := -DNDEBUG -DPROT=1234

#LOCAL_STATIC_LIBRARY变体，所有静态库内容（当几个静态库有循环依赖的时候很有用）
LOCAL_WHOLE_STATIC_LIBRARY := avilib bvilib

#链接标志的可选列表
LOCAL_LDLIBS := -llog

#禁止生产文件中进行缺失符号检查
LOCAL_ALLOW_UNDEFINED_SYMBOLS :

#指定要生成的 ARM 二进制类型
LOCAL_ARM_MODE := arm

#开启 NEON 指令
LOCAL_ARM_NEON := true

#禁用 NX bit 安全特性(Never Execute)
LOCAL_DISABLE_NO_EXECUTE := true

#记录一组编译器标志
LOCAL_EXPORT_CFLAGS := -DENABLE_AUDIO

#与上面类型，仅用于 C++
LOCAL_EXPORT_CPPFLAGS := -DENABLE_AUDIO

#与 LOCAL_EXPORT_CFLAGS 类型用于链接器标志
LOCAL_EXPORT_LDFLAGS := -DENABLE_AUDIO

#允许记录路径集
LOCAL_EXPORT_C_INCLUDES :=

#用于有大量资源或者独立的静态库/动态库的模块，分解命令长度
LOCAL_SHORT_COMMANDS := true

#定义了用于过滤来至 LOCAL_SRC_FILES变量的装配文件的应用程序
LOCAL_FILTER_ASM :=
```

## 0x11 其他构建系统函数宏
```xml
#返回当前目录下的所有子目录下的 Android.mk 构建文件列表
include $(call all-subdir-makefiles)

#放回当前 Android.mk 构建文件的路径
this-makefile

#返回包含当前构建文件的父 Android.mk 构建文件路径
parent-makefile

#和 parent-makefile 一样但用于祖父目录
grand-parent-makefile
```

## 0x12 定义新变量
LOCAL_ 和 NDK_ 开头的为预留构建名称，可使用 MY_ 开头的变量
```xml
MY_SRC_FILES := avilib.c platform_posix.c

LOCAL_SRC_FILES := $(addprefix avilib/, $(MY_SRC_FILES))
```

## 0x13 条件操作
在每个体系结构中包含一个不同的源码文件集
```xml
ifeq ($(TARGET_ARCH), arm)
	LOCAL_SRC_FILES += armonly.c
else
	LOCAL_SRC_FILES += generic.c
endif
```

## 0x14 Application.mk
```xml
#覆盖 Android.mk 构建的模块列表
APP_MODULES := avilib 

#设置二进制文件的优化级别，默认 release
APP_OPTIM := relase / debug

#编译器标志
APP_CLAGS

#编译器标志，作用于 C++ 源文件
APP_CPPLAGS

#从非 jni 子目录下查找 Android.mk 构建文件
APP_BUILD_SCRIPT

#构建系统的二进制文件，默认 armeabi ABI
APP_ABI := all / armeabi mips

#使用不同 STL 实现库，默认 system 库
APP_STL := stlport_shared

#与 LOCAL_CPP_EXTENSIONS 变量相似，表明所有模块都依赖与具体的 C++ 特性，如 RTTI,exceptions
APP_GUNSTL_FORCE_CPP_FEATURES

#与 LOCAL_SHORT_COMMANDS 变量类似
APP_SHORT_COMMANDS
```
