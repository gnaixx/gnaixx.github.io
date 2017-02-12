---
title: DEX文件混淆加密
tags: [android, dex, hidex]
toc: true
date: 2017-02-06 23:24:44
categories: android
description: 现在部分 app 出于安全性(比如加密算法)或者用户体验(热补丁修复bug)会考虑将部分模块采用热加载的形式 Load。所以针对这部分的 dex 进行加密是有必要的，如果 dex 是修复的加密算法，你总不想被人一下就反编译出来吧。当然也可以直接用一个加密算法对 dex 进行加密，Load 前进行解密就可以了，但是最好的加密就是让人分不清你是否加密了。一般逆向过程中拿到一个可以直接反编译成 java 源码的 dex 我们很可能就认为这个 dex 文件是没有加密可以分析的。

---

## 0x00 前言
混淆加密主要是为了隐藏 dex 文件中关键的代码，力度从轻到重包括：静态变量的隐藏、函数的重复定义、函数的隐藏、以及整个类的隐藏。混淆后的 dex 文件依旧可以通过 dex2jar jade 等工具的反编译成 Java 源码，但是里面关键的代码已经看不到了。    
**效果图：**    
![效果图](/blog_images/20170206/compare.png)

## 0x01 dex格式分析
dex 文件格式在上一篇有进行了比较详细的介绍，具体可看[dex文件格式分析](http://gnaixx.cc/2016/11/26/20161126dex-file/)，这里简单的介绍一下整个 dex 文件的布局。  
   
**1.header(dex头部)**    
header 概述了整个 dex 文件的分布情况，包括了：**magic**, **checksum**, **signature**, **file_size**, **header_size**, **endian_tag**, **link**, **map**, **string_ids**, **type_ids**, **proto_ids**, **field_ids**, **method_ids**, **class_defs**, **data**。

- **checksum** 和 **signature** 是校验值，修改后需要对其进行修复
- **string_ids**, **type_ids**, **proto_ids**, **field_ids**, **method_ids** 作为类型数组节区（我瞎起的）保存了不同类型的值
- **class_defs** 存储了类的定义也是我们修改的重点
- **data** 是数据存储区，包括所有的数据

**2.类型数组节区**    
类型数组节区包括了**string_ids**, **type_ids**, **proto_ids**, **field_ids**, **method_ids**。分别表示：字符串，类型，函数签名，属性，函数。每个节区都保存了对应类型数据数组，可以用 010Editor 分析二进制文件数据。    
**属性示例：**     
![field_ids](/blog_images/20170206/field_ids.png)

**3.类定义**    
类定义是修改的重点，这里保存了所有类的结构，也是整个 dex 文件中结构最复杂的部分。其中包括了：静态属性变量、成员数形变量，虚函数，直接函数，静态函数等数据。

## 0x02 实现功能
通过分析 dex 文件格式，现在可以实现的混淆加密主要包括四种：

1. 静态变量隐藏
2. 函数重复定义
3. 函数隐藏
4. 类定义隐藏

四种混淆加密的实现方式都是通过修改 **class_def** 结构体中字段实现的。

```json
{
  "class_def": {
  	"static_values_off": 000,
  	"class_data_off": 001,
  	"class_data": {
  		"direct_methods_size": 001,
  		"virtual_methods_size": 002,
  		"virtual_methods":[
  			{
  				"code_off": 003
  			},
  			{
  				"code_off": 004
  			}
  		]
  	}
  }
}
```

### 1.静态变量隐藏
### 2.函数重复定义
### 3.函数隐藏
### 4.类定义隐藏
## 0x03 数据读取
## 0x04 配置文件
## 0x05 HackPoint格式
## 0x06 校验修复
## 0x07 dex还原(java) 
## 0x08 dex还原(c/c++)
## 0x09 总结