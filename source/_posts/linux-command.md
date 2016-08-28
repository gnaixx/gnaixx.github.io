title: "开发常用的linux命令"
date: 2015-12-16 17:31:13
categories: linux
tags: [java, linux]
toc: true
description: 作为一个Java开发人员，有些常用的Linux命令必须掌握。即时平时开发过程中不使用Linux（Unix）或者mac系统，也需要熟练掌握Linux命令。因为很多服务器上都是Linux系统。所以，要和服务器机器交互，就要通过shell命令。

---

> 毕业后基本都是用mac开发了，渐渐的体会到shell命令的魅力，很多很复杂的操作只要端端的一行shell命令就能解决，大大提高了开发效率。下面给出一些常用的shell命令。


###1.查找文件
<font color=red>find / -name filename.txt</font>　根据名称查找/目录下的filename.txt文件    

<font color=red>find . -name "*.xml" </font>　递归查找当前目录下所有的xml文件    

<font color=red>find . -name "*.xml" |xarg grep "hello world"</font>　 递归查找所有文件内容包含`hello world`的xml文件

<font color=red>grep -H "spring" *.xml</font>　 查找所有包含`spring`的xml文件

<font color=red>find ./ -size 0 |xargs rm -f &</font>　 删除文件大小为0的文件

<font color=red>ls -l |grep ".jar"</font>　 查找当前目录中的所有jar文件

<font color=red>grep "test" d*</font>　 显示所有以d开头的文件中包含test的行

<font color=red>grep "test" aa bb cc</font>　 显示在aa bb cc文件中匹配的test的行

<font color=red>grep "[a-z]\{5\}" aa</font>　 显示aa文件中至少包含五个连续小写字符的字符串的行


###2.查看一个程序是否运行
<font color=red>ps -ef |grep tomcat</font>　 查看所有tomcat的进程

###3.终止程序
<font color=red>kill -9 1292</font>　 终止线程号为1292的进程

###4.查看文件，包含隐藏文件
<font color=red>ls -al</font>　

###5.当前工作目录
<font color=red>pwd</font>　

###6.复制文件
<font color=red>cp source dest</font>　 复制文件

<font color=red>cp -r sourceFolder targetFolder</font>　 递归复制整个文件夹

<font color=red>cp sourceFile remoteUserName@remoteIP:remoteAddr</font>　 远程拷贝

###7.创建目录
<font color=red>mkdir newFolder</font>　

###8.删除目录
<font color=red>rmdir deleteEmptyFolder</font>　 删除空目录

<font color=red>rm -rf deleteFolder</font>　 递归删除目录中所有内容

###9.移动文件
<font color=red>mv /tmp/moveFile /targetFolder</font>　


###10.重命名
<font color=red>mv oldFileName newFileName</font>　

###11.切换用户
<font color=red>su -userName</font>　

###12.修改文件权限
<font color=red>chmod 777 file</font>　 file的权限 -rwxrwxrwx, r表示读、w表示写、x表示可执行

###13.压缩文件
<font color=red>tar -czf test.tar.gz /test1 /test2</font>　

###14.列出压缩文件列表
<font color=red>tar -tzf test.tar.gz</font>　

###15.解压文件
<font color=red>tar -xvzf test.tar.gz</font>　

###16.查看文件
<font color=red>head -n 10 test.txt</font>　 查看文件头10行

<font color=red>tail -n 10 test.txt</font>　 查看文件尾10行

<font color=red>tail -f test.log</font>　 自动显示新增内容，屏幕显示10行，用于查看日志类型文件

###17.查看端口占用情况
<font color=red>netstat -tln |grep 8080 </font>　

####18.查看端口属于哪个程序
<font color=red>lsof -i :8080</font>　

####19.以树状图列出目录内容
<font color=red>tree a</font>　

在Mac系统是没有类似window中的tree命令，找到一条命令可以实现

```shell
find . print | sed -e "s;[^/]*/;|____;g;s;____|; |;g"
```
为了方便使用 写一个alias 到.base_profiel里；当然可以安装tree程序具体可以百度

###20.文件下载
<font color=red>wget http://file.tgz</font>　

<font color=red>curl http://file.tgz</font>　

###21.网络检测
<font color=red>ping www.baidu.com</font>　

###22.远端登录
<font color=red>ssh userName:ip</font>　

###23.打印信息
<font color=red>echo $JAVA_HOME</font>　

###24.liunx命令学习网站
[http://explainshell.com](http://explainshell.com)

























