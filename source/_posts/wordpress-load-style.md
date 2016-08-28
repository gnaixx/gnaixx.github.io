title: "wordpress只有本机能加载样式"
date: 2015-11-11 22:55:21
categories: wordpress
tags: [wordpress]
toc: true
description: 前段时间同事让帮忙搭了一个wordpress，却发现除本机外其他机子无法加载博客的样式
---
wordpress本地可以访问，外网和局域网内其他机子访问无法加载样式   
其实解决办法很简单，修改表`wp_ptions`中`option_name=home`和`option_name= siteurl `的`option_value`值，这个值本来应该是`localhost`改成本机的IP地址就好了。

```sql
update wp_options set option_value='ipaddress' where option_name='siteurl';
update wp_options set option_value='ipaddress' where option_name='home';
```
替换`ipaddress`为你本机IP地址就可以了。