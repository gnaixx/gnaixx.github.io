---
title: 自定义 DexClassLoad(伪)
tags: [android, dex, classload]
toc: true
date: 2017-02-22 14:40:58
categories: android
description: 系统 API 不管 DexClassLoader 还是 PathClassLoader 都只支持文件路径参数，所以在加载 dex 文件的时候必须生成一个缓存文件，而且是正常格式的 dex 文件。自定义的目的就是去掉这个缓存的过程，简单来讲就是让 DexClassLoader 支持 byte 数组参数，以内存形式加载。为什么加个伪，因为只实现了 Android4.0 系统到 Android4.4 系统😞，对于其他版本的系统算不上真正意义上的自定义。

---
## 0x00 前言

## 0x01 load 流程

## 0x02 源码分析

## 0x03 代码实现

## 0x04 总结