title: "\"**游\"强制更新破解"
date: 2016-06-19 21:46:58
categories: android
tags: [android, crack, burp]
toc: true
description: 好久没有更新博客了，因为一些特操蛋的事情，一直没有心思写。上两周终于把《Android 软件安全与逆向分析》 看完了，然后就老是想找个 apk 练练手，但是水平太差，稍微加密就啃不动了。刚好遇到了这个 apk 习惯性逆向分析。

---

##0x00 前言
前几天客户有提到一个作弊工具，所以就想下载体验一下。结果下载的应该是一个较早的版本需要强制更新到最新版本，不然不能运行。    
**更新提示：**    
![1.png](https://gnaixx.github.io/blog_images/txy/1.png)    
所以就想破解一下他的强制更新练练手，因为感觉不难。

##0x01 分析Java代码
利用 `jadx-gui` 反编译 apk，`jadx-gui` 这个工具还是挺好用的。一条命令就可以看到反编译后的 Java 代码，还有资源文件。

###AndroidManifest.xml分析
`Application` 是 Android 程序的主入口，也是分析 apk 的开始点。而且通过运行程序也可以发现。跳出的升级的 Dialog 是在程序运行后第二个页面跳出的，第一个页面为启动页面。所以重点分析对象就是 `Application` 到第二个页面。    
`AndroidManifest` 查找 `application` 标签看有没有自定义 `Application`
![2.png](https://gnaixx.github.io/blog_images/txy/2.png)

过滤 `android.intent.action.MAIN` 查找第一个页面。   
![3.png](https://gnaixx.github.io/blog_images/txy/3.png)

###Activity分析
自定义的 `Application` 中没有什么信息，只是做了一些配置的初始化和第三方 SDK 初始化的。    

####SplashActivity
`SplashActivity` 是 apk 的第一个页面也就是启动页面，因为代码没有做混淆处理，其中我们还是可以发现一些关键的代码。    
![4.png](https://gnaixx.github.io/blog_images/txy/4.png)   
![5.png](https://gnaixx.github.io/blog_images/txy/5.png)   
变量 `isUpdate` 是关键点，也可以发现这个页面停了100ms后会跳转到 `MainActivity` 页面。但是这个页面只是对变量 `isUpdate` 进行了判断赋值没有进行升级的处理。

####MainActivity
`MainActivity` 真正的主页面，查看 `onCreate`方法。    
![6.png](https://gnaixx.github.io/blog_images/txy/6.png)    
当时只看到了第二个标注的代码，心想这不是改下 `if-else` 的事吗。（下面会将第一个标注）

##0x02 修改Smali代码

`apktool` 反编译 APK，没有做过加固处理直接。定位到 `MainActivity.onCreate` 方法，修改`if-else`逻辑，将`if-eqz v3, :cond_3`改为以下代码：    
![7.png](https://gnaixx.github.io/blog_images/txy/7.png)  
原本为 `isUpdate` 为 `true`时，执行 `isUpdateAlert()` 方法。改为 `false`时执行。 顺便还加了一个 Log，回编安装运行。    
![8.png](https://gnaixx.github.io/blog_images/txy/8.png)    
结果发现自己还是太年轻，没有绕过升级。虽然Log是有打印了，证明代码是没有问题的，但是在升级的 Dialog 还是出现了，并且之前还有一个 Dialog。    
![9.png](https://gnaixx.github.io/blog_images/txy/9.png)   
那天研究太晚了就睡觉去了。

##0x03 重新分析代码逻辑
第二天想想肯定是自己哪里分析错了，又重新看了一遍代码。首先看下之前认为是改变逻辑后执行的函数 `isUpdateAlert()`:    
![10.png](https://gnaixx.github.io/blog_images/txy/10.png)     
可以看到当用户点击确定的时候只改变了 `SplashActivity.isUpdate` 的值，没有其他操作了。再查找一下 `R.string.str_reminder` 的值在目录 `res/values-zh/strings.xml`:     
![11.png](https://gnaixx.github.io/blog_images/txy/11.png)     
好吧事实证明自己还太年轻啊。      

###分析变量 isUpdate 的作用
所以可以先回到 `SplashActivity` 看看变量是怎么赋值的。     
![12.png](https://gnaixx.github.io/blog_images/txy/12.png)    
可以看到 `isUpate` 的值是根据 **Initialized** 文件中保存的值跟当前APK的版本做比较得到的值。    
![13.png](https://gnaixx.github.io/blog_images/txy/13.png)     
方法 `setInitFlag` 则是将当前版本写入 **Initialized**。这样看来貌似 `isUpdate` 不可能为 `true`了。 再看下什么地方调用了这两个函数：     
![14.png](https://gnaixx.github.io/blog_images/txy/14.png)     
到这里可以明白了，**Initialized** 文件是写入当前版本号，但是当apk更新后版本号变了，跟 **Initialized** 中的版本号就不一样了。所以  `isUpdate` 记录的是当前 APK 是不是已经更新过了并且第一次打开，而不是当前 APK 是不是需要更新。

##0x04 burp抓包分析
通过重新分析可以断定之前的修改是错误的了，但是既然 APK 需要判断是否需要更新那么就必须有网络请求，burp 该上场了，挂上代理打开APK：   
![15.png](https://gnaixx.github.io/blog_images/txy/15.png)    
这里请求成功的只有三个 HTTP 请求（百度地图的忽略),HTTP 还省了导入证书了。一个一个分析请求的 response 。抓包这个技能还是必须掌握的：     
![16.png](https://gnaixx.github.io/blog_images/txy/16.png)     
知道了哪个接口调用的那就简单了，直接找出这个接口的类：    
![17.png](https://gnaixx.github.io/blog_images/txy/17.png)   
用的友盟的更新模块，所以我上面忽略的才是真正的更新代码。 

```java
UmengUpdateAgent.setUpdateOnlyWifi(false);
UmengUpdateAgent.forceUpdate(this);
```      
再看 `com.umeng.update.UpdateResponse` 这个类的 `a(JSONObject)` 方法:      
![18.png](https://gnaixx.github.io/blog_images/txy/18.png)    
对比返回的 response （burp复制出来中文乱码了,看key就好）    

```json
{
  "update": "Yes",
  "version": "12.1.2",
  "path": "http://au.apk.umeng.com/uploads/apps/5386ea3b6c738f0de0000025/_umeng_%40_188_%40_09e2ea839754f61d4fb24ddb96a638bf.apk",
  "origin": "",
  "update_log": "å¤©ä¸æ¸¸12.1.2\r\n1.ä¿®å¤å°ç±³5ä¸è½ä½¿ç¨çé®é¢\r\n2.éä½çµéæ¶èãæé«ç¨³å®æ§åå¼å®¹æ§\r\n3.æ´æ°ä¼æ¸é¤12.0ä»¥åçæ¬æ°æ®\r\n4.æ´æ°å®æåè¯·éå¯ææºï¼ä»¥ä½¿æ´æ°çæï¼ï¼ï¼\r\n5.å¦éå°ä¸è½ä½¿ç¨çæåµï¼è¯·å°âé®é¢åé¦âä¸­æè¿°é®é¢å¹¶çä¸æ¨çèç³»æ¹å¼",
  "proto_ver": "1.4",
  "delta": false,
  "new_md5": "09e2ea839754f61d4fb24ddb96a638bf",
  "size": "18278969",
  "patch_md5": "",
  "target_size": "18278969",
  "display_ads": false
}
``` 
所以这个类就是处理返回值的地方了。这不是修改一下 `Yes`字符串就完了，哈哈哈哈哈哈。

##0x05 再次修改Smali
添加了Log 将 `Yes` 改为 `No`:     

![19.png](https://gnaixx.github.io/blog_images/txy/19.png)   

回编 安装 运行，终于不再提示更新了。。。。。。。


##0x06 总结
逆向分析真是一个细致活啊，稍微有点忽略就走了弯路。另外对于 release 的版本的APK 最好可以进行 proguard 混淆和加固，现在有很多第三方加固都提供免费的服务了，做了这两步我想就能挡住很多像我这样的菜鸟了。     
很久没更新博客，这篇也写了好几天才写好。。。。。。。。。
   