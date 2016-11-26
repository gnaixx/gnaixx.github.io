---
title: Android.mk 配置
tags: [android, ndk, makefile]
toc: true
date: 2016-11-06 21:05:48
categories: android
description: Android.mk 的可配置参数很多，但是其实在自己开发中用到的其实比较简单比较少。但是有必要知道ndk编译中可以使用那些参数。

---

## 0x00 LOCAL_PATH
根据 Android 构建系统要求，Android.mk 文档必须以 LOCAL_PATH 变量的定义开头。    
`LOCAL_PATH := $(call my-dir)`     
Android 构建系统利用 LOCAL_PATH 来定位源文件。因为将改变量设置为硬编码值不合适，所以 Android 构建系统提供了一个名为 my-dir 的宏功能。通过将该变量设置为 my-dir 宏功能的返回值，可以将其放在当前目录下。

## 0x01 CLEAR_VARS
Android 构建系统将 CLEAR_VARS 变量设置为 clear-vars.mk 片段的位置。包含 Makefile 片段可以清除除了 LOCAL_PATH 以外的LOCAL_name 变量，例如 LOCAL_MODULE 与 LOCAL_SRC_FILE 等。    
`Include $(CLEAR_VARS)`    
这样做是因为 Android 构建系统可以在单次执行中解析多个构建文件和模式定义，LOCAL_<name> 是全局变量。清除它们可以避免冲突，每一个原生组件被称为一个模块。

## 0x02 LOCAL_MODULE
改变了是用来给这些模块设定一个唯一的名称。下面的代码将该模块的名称设为 hello-jni:    
`LOCAL_MODULE := hello-jni`    
因为模块名称也被用于给构建过程的所生成的文件命名，所以构建系统给文件添加适当的前缀和后缀。本例中， hello-jni 模块会生成一个共享文件且构建系统会将它命名为 libhello-jni.so。

## 0x03 LOCAL_SRC_FILES
该变量用来建立和组装这个模块的源文件列表。     
`LOCAL_SRC_FILES := hello-jni.c`     
这里，hello-jni 模块只有一个源文件生成，而 LOCAL_SRC_FILES 变量可以包含用空格分开的多个源文件名。

## 0x04 BUILD_SHARED_LIBRARY
为了建立可供主应用程序使用的模块，必须将该模块变成共享库。Android NDK 构建系统将 BUILD_SHARED_LIBRARY 变量设置成 build_shared_libray.mk 文件保存的位置。改 Makefile 片段包含了将源文件构建和组装成共享库的必要过程。    
`include $(BUILD_SHARED_LIBRARY)`   
