title: "TextPaint绘制文字"
date: 2015-09-25 20:12:03
categories: android
tags: [android, TextPaint]
toc: true
description: 这两天在写2048的游戏，涉及到了绘制文字。所以用到了TextPaint，用它可以很方便的进行文字的绘制。
---

TextPaint是paint的子类，一般情况下遇到绘制文字的需求时可以用它提供的方法。

##FontMetris
FontMetris是Paint的内部类，作为"字体测量"。它定义了top、ascent、descent、bottom、leading五个成员变量。
