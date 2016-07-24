title: "Android抽屉菜单的实现-DrawerLayout(上)"
date: 2015-11-09 22:58:25
categories: android
tags: [android, material design, DrawerLayout]
toc: true
description: 相信现在的Android开发没有不知道Material Design的，但是在Android开发中Material Design并不是很容易实现的。不过在谷歌2015 I/O 大会时，谷歌宣布了一个今年最让人兴奋的支持库，名叫 Android Design Support Library，在这个单独的 library 里提供了一堆有用的材料设计 UI 组件。

---
>相信现在的Android开发没有不知道`Material Design`的，但是在Android开发中`Material Design`并不是很容易实现的。不过在谷歌2015 I/O 大会时，谷歌宣布了一个今年最让人兴奋的支持库，名叫 **Android Design Support Library**，在这个单独的 library 里提供了一堆有用的材料设计 UI 组件。

最近一段深深的被Material Design所吸引，厌倦了iPhone风格，特别是类iPhone风格的Android App。所以尝试开始学习写一些Material Design风格Demo。

###抽屉菜单
Android的抽屉菜单很早就有应用使用了，不过自定义需要重写大量的代码。google官方的**Support v4**包中DrawerLayout可以轻松的实现该功能。[Demo源码地址DrawerrActivity页面](https://github.com/gnaix92/demo-material)。

###新建项目添加依赖
新建一个Android应用并添加**Support v4**，当然你也可以直接添加**Design Support Library**因为它已经包含了 **Support v4** 和 **AppCompat v7**。
`build.gradle`代码：

```java
    compile 'com.android.support:design:23.1.0'
    compile 'com.android.support:support-v4:23.1.0'
```

###页面布局
`DrawerLayout`的使用非常简单，它可以包含三个子控件，依次为：   

- 主页面
- 左划菜单
- 右滑菜单

``` xml
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/drawer_layout"
    android:layout_height="match_parent"
    android:layout_width="match_parent"
    tools:context=".TestActivity">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/app_name"/>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:background="@color/white"
        android:orientation="vertical">
    </LinearLayout>
    
    <!-- 这里没有实现右滑菜单-->

</android.support.v4.widget.DrawerLayout>
```

需要注意的是左划菜单需要设置`android:layout_gravity`为`start/left`。右划菜单则为`end/right`。

###设置toolbar关联
写好xml其实已经可以最基础的左滑功能，但是我们需要做成下面的效果让他和toolbar关联。   
![](../../../../blog_images/material-darwer/1.gif)

需要设置DrawerLayout的监听器`ActionBarDrawerToggle`。

```java
 	DrawerLayout drawerLayout;
    ActionBarDrawerToggle drawerToggle;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_drawer);

        initInstances();
    }

    private void initInstances(){
        drawerLayout = (DrawerLayout) findViewById(R.id.drawer_layout);
        drawerToggle = new ActionBarDrawerToggle(DrawerActivity.this, drawerLayout, R.string.open, R.string.close);
        drawerLayout.setDrawerListener(drawerToggle);

        getSupportActionBar().setDisplayHomeAsUpEnabled(true);
        getSupportActionBar().setHomeButtonEnabled(true);
    }
```
设置页面更新时toolbar图标变化

```java
	@Override
    public void onPostCreate(Bundle savedInstanceState) {
        super.onPostCreate(savedInstanceState);
        drawerToggle.syncState();
    }

    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
        drawerToggle.onConfigurationChanged(newConfig);
    }
```

最后不要忘记了在`onOptionsItemSelected`中添加

```java
if (drawerToggle.onOptionsItemSelected(item))
	return true;
```
被这坑了好久，我说怎么按图标抽屉就是出不来。

<font color=red>**PS:**</font>
Demo中还实现了另外一种设计改天讲，睡觉！
