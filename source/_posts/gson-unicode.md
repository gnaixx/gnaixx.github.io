title: "避免Gson使用时将一些字符自动转换为Unicode转义字符"
date: 2015-11-06 23:01:11
categories: android
tags: [Gson, Unicode]
toc: true
description: 今天在排查客户反馈的问题时候发现他们穿过来的数据有异常，因为我们获取的数据会添加一层Base64编码，而Base64会用`=`符号填充，但是我们发现`=`被转移为`\u003d`
---

今天在排查客户反馈的问题时候发现他们穿过来的数据有异常，因为我们获取的数据会添加一层Base64编码，而Base64会用`=`符号填充，但是我们发现`=`被转移为`\u003d`。经过排查可能是客户端利用Gson产生字符串导致部分字符被自动转换为Unicode字符。

----
####例如：
```java
{"s":"\u003c"}
```
我只想简单的打印成这样

```java
{"s":"<"}
```

####解决方案：

我只需要 <font color=red>`disable HTML escaping`</font>.

```java
Gson gson = new GsonBuilder().disableHtmlEscaping().create();
```