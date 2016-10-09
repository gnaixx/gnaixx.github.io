title: "android中发生no peer certificate异常"
date: 2015-11-12 23:52:54
categories: android
tags: [android, certificate]
toc: true
description: 在互联网安全通信方式上，目前用的最多的就是https配合ssl和数字证书来保证传输和认证安全了,所以基本每一个开发者都会遇到SSLPeerUnverifiedException问题。这篇blog主要讲下Android中产生证书问题的原因和解决方案

---

###0x00 问题
---
相信在使用https协议开发中很多人都会遇到证书异常的情况：
<font color=red>**javax.net.ssl.SSLPeerUnverifiedException: No peer certificate**</font>   

在解决这个问题前我们必须先了解一下https和ssl
####什么是HTTPS
https是在http(超文本传输协议)基础上提出的一种安全的http协议，因此可以称为安全的超文本传输协议。http协议直接放置在TCP协议之上，而https提出在http和TCP中间加上一层加密层。从发送端看，这一层负责把http的内容加密后送到下层的TCP，从接收方看，这一层负责将TCP送来的数据解密还原成http的内容。
####什么是SSL/TLS
SSL全称是Secure Sockets Layer，安全套接字层，它是由网景公司（Netscape）设计的主要用于Web的安全传输协议，目的是为网络通信提供机密性、认证性及数据完整性保障。如今，SSL已经成为互联网保密通信的工业标准。

SSL最初的几个版本（SSL 1.0、SSL2.0、SSL 3.0）由网景公司设计和维护，从3.1版本开始，SSL协议由因特网工程任务小组（IETF）正式接管，并更名为TLS（Transport Layer Security），发展至今已有TLS 1.0、TLS1.1、TLS1.2这几个版本。

如TLS名字所说，SSL/TLS协议仅保障传输层安全。同时，由于协议自身特性（数字证书机制），SSL/TLS不能被用于保护多跳（multi-hop）端到端通信，而只能保护点到点通信。

SSL/TLS协议能够提供的安全目标主要包括如下几个：

- 认证性——借助数字证书认证服务器端和客户端身份，防止身份伪造

- 机密性——借助加密防止第三方窃听

- 完整性——借助消息认证码（MAC）保障数据完整性，防止消息篡改

- 重放保护——通过使用隐式序列号防止重放攻击

为了实现这些安全目标，SSL/TLS协议被设计为一个两阶段协议，分为握手阶段和应用阶段：

握手阶段也称协商阶段，在这一阶段，客户端和服务器端会认证对方身份（依赖于PKI体系，利用数字证书进行身份认证），并协商通信中使用的安全参数、密码套件以及MasterSecret。后续通信使用的所有密钥都是通过MasterSecret生成。

在握手阶段完成后，进入应用阶段。在应用阶段通信双方使用握手阶段协商好的密钥进行安全通信。  
[专业级的解释](http://drops.wooyun.org/tips/6002)就看懂了上面这部分

###0x01 问题分析
---     
分析问题最重要的是要把https工作流程给搞清楚    
**
C: Client发送一个request（包括密码套件，SSL/TLS版本，压缩算法）  
S: Server把支持的加密算法等以证书的形式返回（包括CA颁发机构和加密公钥，SSL使用的证书通常是X.509类型）   
C: Client获取证书后，验证证书颁发机构，并通过与本地已有的数字证书验证Server证书的真实性。如果其中有验证失败的情况，发出警告并断开连接   
C: 验证通过，产生一份消息，消息处理后将用作对称加密的密钥。将这份消息用Server的公钥进行加密。把这份加密的消息（ClientKeyExchange)发送给Server
S: Server用自己的私钥将ClientExchange中的消息解密出来作为对称加密的密钥，这时双方已经有了一套安全的加密方式了  
之后双方传输数据可以使用该密钥进行加密然后对话了  
**  
<font color=red>ps:如果不清楚公钥、私钥、对称加密这些的可以去[google](https://www.google.com)</font>    
清楚了流程不难定位出在Client验证证书时出现了异常。


###0x02 原因及解决方案
---

####证书的颁发者不在“根受信任的证书颁发机构列表”中
出现这种情况大多数是自定义的证书，证书没有通过权威机构认证。因为证书需要购买并且有年限，所以对于很多想用https协议的个人开发者来说自定义证书是比较合算的。

#####导入证书
在客户端设备中导入自定义证书，但是这种操作是客户做的不是开发者改考虑的

#####信任全部证书
信任证书其实就是把校验证书这一步骤给省略了,源码在之前的一篇blog中有提到**[Android http/https网络请求](/2015/05/20/http-https/)**   
虽然信任全部证书可以解决这一问题并且很多大公司也这么搞过，但是这样做的风险还是比较大的，容易遭受SSL中间人攻击。乌云平台上有很多该漏洞还包括很多知名的app。**[Android证书信任问题与大表哥](http://drops.wooyun.org/tips/3296)**

#####下载证书保存到本地
这种方式是比较保险的，当然如果是你自己的app可以先将证书保存在assets，而不需要从网上拉取

* 将证书下载保存到Android的assets目录下
* 导入证书
* 添加证书为信任

```java
    String requestHTTPSPage(String mUrl) {
        InputStream ins = null;
        String result = "";
        try {
            ins = context.getAssets().open("app_pay.cer"); //下载的证书放到项目中的assets目录中
            CertificateFactory cerFactory = CertificateFactory
                    .getInstance("X.509");
            Certificate cer = cerFactory.generateCertificate(ins);
            KeyStore keyStore = KeyStore.getInstance("PKCS12", "BC");
            keyStore.load(null, null);
            keyStore.setCertificateEntry("trust", cer);
 
            SSLSocketFactory socketFactory = new SSLSocketFactory(keyStore);
            Scheme sch = new Scheme("https", socketFactory, 443);
            HttpClient mHttpClient = new DefaultHttpClient();
            mHttpClient.getConnectionManager().getSchemeRegistry()
                    .register(sch);
 
            BufferedReader reader = null;
            try {
                Log.d(TAG, "executeGet is in,murl:" + mUrl);
                HttpGet request = new HttpGet();
                request.setURI(new URI(mUrl));
                HttpResponse response = mHttpClient.execute(request);
                if (response.getStatusLine().getStatusCode() != 200) {
                    request.abort();
                    return result;
                }
 
                reader = new BufferedReader(new InputStreamReader(response
                        .getEntity().getContent()));
                StringBuffer buffer = new StringBuffer();
                String line = null;
                while ((line = reader.readLine()) != null) {
                    buffer.append(line);
                }
                result = buffer.toString();
                Log.d(TAG, "mUrl=" + mUrl + "\nresult = " + result);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if (reader != null) {
                    reader.close();
                }
            }
        } catch (Exception e) {
            // TODO: handle exception
        } finally {
            try {
                if (ins != null)
                    ins.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return result;
    }
```

####证书过期
这种情况其实很简单不外乎更新证书，当然你也可以信任全部证书（不建议这么干）

####客户端系统时间问题
这个问题是自己在开发的时候遇到的，困扰了很久。因为没有深入了解证书校验的具体流程所以还是被坑了很久。Client在校验证书的时候时间戳也是一项。如果Client的时间和Server时间相差比较大的话（或者在证书有效期时间外）验证就会失败。这其实是SSL/TLS的重放保护。    
解决方法其实很简单：把客户端的时间设置成自动时间和日期（使用网络提供的时间），但是这不是开发者能决定的。<font color=red>ps:信任全部证书可以解决</font>  
测试了很多app都会有这种情况。

####权限缺失
这种情况是网上搜索的时候看到的，并没有实际遇到过。  
**解决方案:**   
在AndroidManifest.xml中添加下列权限    

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.WRITE_APN_SETTINGS"/>
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE"/>
```

###0x03总结
---

https中发生<font color=red>**javax.net.ssl.SSLPeerUnverifiedException: No peer certificate**</font> 的情况还是比较多的，解决问题的关键是先搞清楚https自己也是因为不是了解导致坑了很久。还有一点，多google。