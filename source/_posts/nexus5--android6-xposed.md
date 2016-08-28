title: "nexus5 刷 Android6.0+Xposed"
date: 2016-07-24 19:14:54
categories: android
tags: [xposed nexus5]
toc: true
description: 这几天把手机弄坏了好几次，每次都需要重新刷一遍。刷系统+root+刷recovery+安装xposed。

---
> 不得不说现在的刷机工具都做的方便快捷，只需要简单的几天命令就解决了。以前用windows刷的时候还需要装一堆驱动软件啥的。

## 0x00 刷入Android6.0系统
###下载镜像
官方的镜像刷机真是简单快捷，所有操作他都写好了脚本，你要做的就是运行它。当然前提是你已经安装好了ADB。下载的Android SDK里面就已经集成了，这里就不说了。    
[官方镜像下载地址](https://developers.google.com/android/nexus/images)

###具体步骤
1. 解压下载的镜像，其中刷机脚本就是`flash-all.sh`   
![1.png](http://gnaix92.github.io/blog_images/nexus/1.png) 

2. 关机，进入fastboot模式（关机状态下同时按住：音量下，电源键）
3. 连接电脑，运行`flash-all.sh`脚本(需要一点时间)    
![2.png](http://gnaix92.github.io/blog_images/nexus/2.png) 

##0x01 一键Root
使用CF-Auto-Root一键root 安卓6.0.1或（安卓n第一个预览版）。此办法的好处是无需很多技巧，但是坏处是包更新不及时，比如到了安卓6.1使用该包就会刷入老的内核版本。   
[root 工具包](https://download.chainfire.eu/363/CF-Root/CF-Auto-Root/CF-Auto-Root-hammerhead-hammerhead-nexus5.zip)

刷root的前记得开启usb调试模式。打开fastboot模式，操作也非常简单，运行`root-mac.sh`就好了。

##0x02 刷入第三方recovery twrp
刷入第三方recovery有很多好处，最大的好处就是方便刷其他的rom，同时还可以通过其install功能来实现root和刷入一些模块等，同时twrp还提供了很多诸如备份啊，文件夹管理等功能，是玩机的必备工具之一。    
找到适配您的nexus设备的twrp(您可以在https://twrp.me/Devices/ 输入相关设备名即可找到最新的)     
[twrp for nexus 5](https://dl.twrp.me/hammerhead/twrp-2.8.7.1-hammerhead.img)

###具体步骤
1. 关机，进入fastboot模式（关机状态下同时按住：音量下，电源键）
2. .刷入twrp的img（使用fastboot flash recovery twrp.img）  
![3.png](http://gnaix92.github.io/blog_images/nexus/3.png) 
3. 刷入成功后，可以在fastboot状态下，用音量键来选择"recovery mode"来进入twrp

##0x03 刷入Xposed框架
下载Xposed框架：[http://dl-xda.xposed.info/framework/sdk23/arm/](http://dl-xda.xposed.info/framework/sdk23/arm/)
我这里是下载最新的v86-sdk23版本。 
###具体步骤 
1. 将压缩包复制到sdcard上：
![4.png](http://gnaix92.github.io/blog_images/nexus/4.png) 

2. 点击twrp上的install，选择刚刚导入的压缩包，重启就OK了。

3. 最后安装[XposedInstaller](http://forum.xda-developers.com/attachment.php?attachmentid=3383776&d=1435601440)。

