title: Ubuntu 学习笔记

date: 2015-02-01

categories: Ubuntu

tags: ubuntu, note
---

最近一直在习惯Linux系统，在这里记录下使用期间遇到的问题。 


##如何在Ubuntu系统下打开.psd格式文件

----------

####直接使用Open Office打开
存在问题：效果不好锯齿明显

####通过GIMP软件
网上有人提到gimp可以打开psd文件，安装方法很简单。通过软件中心安装就可以。
但是我试了都是打开失败， 具体原因不清楚。  

####最好的方法安装gwenview
前面两个方法都有明显的缺陷，gwenview相对来说是比较好了。但是他只是一个查看工具不能修改。  
安装方法：`sudo apt-get install gwenview`