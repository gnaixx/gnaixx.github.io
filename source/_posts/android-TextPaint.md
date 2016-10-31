title: "TextPaint绘制文字"
date: 2015-09-25 20:12:03
categories: android
tags: [android, TextPaint]
toc: true
description: 这两天在写2048的游戏，涉及到了绘制文字。所以用到了TextPaint，用它可以很方便的进行文字的绘制。
---

TextPaint是paint的子类，一般情况下遇到绘制文字的需求时可以用它提供的方法。

##0x00 FontMetris

FontMetris是Paint的内部类，作为"字体测量"。它定义了top、ascent、descent、bottom、leading五个成员变量。   

```java
/**
     * Class that describes the various metrics for a font at a given text size.
     * Remember, Y values increase going down, so those values will be positive,
     * and values that measure distances going up will be negative. This class
     * is returned by getFontMetrics().
     */
    public static class FontMetrics {
        /**
         * The maximum distance above the baseline for the tallest glyph in
         * the font at a given text size.
         */
        public float   top;
        /**
         * The recommended distance above the baseline for singled spaced text.
         */
        public float   ascent;
        /**
         * The recommended distance below the baseline for singled spaced text.
         */
        public float   descent;
        /**
         * The maximum distance below the baseline for the lowest glyph in
         * the font at a given text size.
         */
        public float   bottom;
        /**
         * The recommended additional space to add between lines of text.
         */
        public float   leading;
    }
```
##0x01 变量含义

下面两张图可以很好的说明这几个变量的含义  
![textpaint](https://gnaixx.github.io/blog_images/textpaint/textpaint_1.png)
![textpaint](https://gnaixx.github.io/blog_images/textpaint/textpaint_2.png)
   
- Baseline：是基线，在Android中，文字的绘制都是从Baseline处开始的
- Ascent：Baseline往上至字符“最高处”的距离我们称之为ascent
- Baseline往下至字符“最低处”的距离我们称之为descent
- Leading：则表示上一行字符的descent到该行字符的ascent之间的距离
- Top,Bottom：TextView在绘制文本的时候总会在文本的最外层留出一些内边距，如图2"A"顶上字符。

##0x02 代码验证

```java
private static final String TEXT = "ap卡了ξτβбпшㄎㄊěǔぬも┰┠№＠↓";
```
```java
 @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        mTextPaint.setTextSize(50);  
        mTextPaint.setColor(Color.BLACK);  
        
        FontMetrics fontMetrics = mTextPaint.getFontMetrics();  
        Log.d("Aige", "ascent：" + fontMetrics.ascent);  
        Log.d("Aige", "top：" + fontMetrics.top);  
        Log.d("Aige", "leading：" + fontMetrics.leading);  
        Log.d("Aige", "descent：" + fontMetrics.descent);  
        Log.d("Aige", "bottom：" + fontMetrics.bottom);  
        
        mTextPaint.clearShadowLayer();
        canvas.drawText(TEXT, 0, Math.abs(fontMetrics.top), mTextPaint);
    }
```
**结果：**

```java
打印的Log：
ascent：-46.38672
top：-52.807617
leading：0.0
descent：12.207031
bottom：13.549805
```
<font color=red><b>注：Baseline上方的值为负，下方的值为正</b></font>

##0x03 FontMetrics中的变量和文字的size，typeface有关

从代码中我们可以看到一个很特别的现象，在我们绘制文本之前我们便可以获取文本的FontMetrics属性值，也就是说我们FontMetrics的这些值跟我们要绘制什么文本是无关的，而仅与绘制文本Paint的size和typeface有关。当你改变了paint绘制文字的size或typeface时，FontMetrics中的top、bottom等值就会发生改变。如果我们仅仅更改了文字，这些值是不会发生任何改变的。

##0x04 绘制居中屏幕文字
**代码**

```java
@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        mTextPaint.setTextSize(50);
        mTextPaint.setColor(Color.BLACK);

        // 计算Baseline绘制的起点X轴坐标 ，计算方式：画布宽度的一半 - 文字宽度的一半
        int baseX = (int) (canvas.getWidth() / 2 - mTextPaint.measureText(TEXT) / 2);

        // 计算Baseline绘制的Y坐标 ，计算方式：画布高度的一半 - 文字总高度的一半
        int baseY = (int) ((canvas.getHeight() / 2) - ((mTextPaint.descent() + mTextPaint.ascent()) / 2));

        // 居中画一个文字
        canvas.drawText(TEXT, baseX, baseY, mTextPaint);

        mPaint.setColor(Color.RED);
        mPaint.setStrokeWidth(2);
        // 为了便于理解我们在画布中心处绘制一条中线
        canvas.drawLine(0, canvas.getHeight() / 2, canvas.getWidth(), canvas.getHeight() / 2, mPaint);
    }
```

我们计算了x坐标和y坐标。

x坐标的计算方法是（屏幕宽度-文字宽度）/2，如果文字宽度比屏幕宽度长得到的就是负数，如果文字宽度比屏幕宽度短，得到的就是正数，这个很容易理解；

y坐标的的计算方式是（屏幕高度-文字高度）/2，这里的文字高度用的是：descent+ascent（忽略了音标）。

**结果** 

<img width=220px height=350px src="https://gnaixx.github.io/blog_images/textpaint/textpaint_3.png"><img>