title: Activity中获取View高度与宽度
date: 2015-02-05
categories: Android
tags: [View, android, activity]
---
在activity中可以调用`View.getWidth、View.getHeight()、`
`View.getMeasuredWidth()、View.getgetMeasuredHeight()`来获得某个view的宽度或高度，但是在`onCreate()、onStrart()、onResume()`方法中会返回0，原因如下：

- 当前`activity`所代表的界面还没显示出来没有添加到`WindowPhone`的`DecorView`上
- 要获取的`view`没有被添加到`DecorView`上
- 该View的`visibility`属性为`gone` 
- 该View的`width`或`height`真的为0
   
只有上述条件都不成立时才能得到非0的width和height  
所以要想获取到的width和height为真实有效的 则有以下方法
    



1.在该View的事件回调里使用这时候该View已经被显示即被添加到`DecorView`上如点击事件触摸事件焦点事件等


```java
View view=findViewById(R.id.tv);
view.setOnClickListener(new OnClickListener() {		
	@Override
	public void onClick(View v) {
		int width = v.getWidth();
	}
});
```

2.在activity被显示出来时即添加到了`DecorView`上时获取宽和高如`onWindowFocusChanged()`回调方法

```java
@Override
public void onWindowFocusChanged(boolean hasFocus) {
	View iv1 = findViewById(R.id.iv1);
	View iv2=findViewById(R.id.iv2);
	String msg1="iv1' width:"+iv1.getWidth()+" height:"+iv1.getHeight()+"  measuredWidth:"+iv1.getMeasuredWidth()+"measuredHeight:"+iv1.getMeasuredHeight();
	String msg2="iv2' width:"+iv2.getWidth()+" height:"+iv2.getHeight()+"  measuredWidth:"+iv2.getMeasuredWidth()+"measuredHeight:"+iv2.getMeasuredHeight();
	i("onWindowFocusChanged() "+msg1);
	i("onWindowFocusChanged() "+msg2);
    super.onWindowFocusChanged(hasFocus);
}
```

3.在`onResume`方法后开线程300毫秒左右后获取宽和高，因为`onResume`执行完后300毫秒后界面就显示出来了  

当然第2种和第3种方法要保证获取宽高的view是在setContentView时设进去的View或它的子View
```java
view.postDelayed(new Runnable() {			
	@Override
	public void run() {
		View iv1 = findViewById(R.id.iv1);
		View iv2=findViewById(R.id.iv2);
		String msg1="iv1' width:"+iv1.getWidth()+" height:"+iv1.getHeight()+"  measuredWidth:"+iv1.getMeasuredWidth()+"measuredHeight:"+iv1.getMeasuredHeight();
		String msg2="iv2' width:"+iv2.getWidth()+" height:"+iv2.getHeight()+"  measuredWidth:"+iv2.getMeasuredWidth()+"measuredHeight:"+iv2.getMeasuredHeight();
		i("onWindowFocusChanged() "+msg1);
		i("onWindowFocusChanged() "+msg2);
	}
}, 300);
```


4.在`onCreate()`或`onResume()`等方法中需要获取宽高时使用`getViewTreeObserver().addOnGlobalLayoutListener()`来添为view加回调在回调里获得宽度或者高度获取完后让view删除该回调
```java
view.postDelayed(new Runnable() {		
	@Override
	public void run() {
		View iv1 = findViewById(R.id.iv1);
		View iv2=findViewById(R.id.iv2);
		String msg1="iv1' width:"+iv1.getWidth()+" height:"+iv1.getHeight()+"  measuredWidth:"+iv1.getMeasuredWidth()+"measuredHeight:"+iv1.getMeasuredHeight();
		String msg2="iv2' width:"+iv2.getWidth()+" height:"+iv2.getHeight()+"  measuredWidth:"+iv2.getMeasuredWidth()+"measuredHeight:"+iv2.getMeasuredHeight();
		i("onWindowFocusChanged() "+msg1);
		i("onWindowFocusChanged() "+msg2);
	}
}, 300);
```