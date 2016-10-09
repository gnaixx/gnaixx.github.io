title: "Android http/https网络请求"
date: 2015-05-20 11:21:02
categories: android
tags: [android, http/https]
toc: true
description: Android中实现网络请求主要通过HttpClient，网络请求又分为get和post两种。HttpClient支持http和https，http实现相对简单。https是安全的http协议，在使用的时候必须考虑SSL和数字证书的问题。

---
Android中实现网络请求主要通过HttpClient，网络请求又分为get和post两种。HttpClient支持http和https，http实现相对简单。[https](http://www.cnblogs.com/P_Chou/archive/2010/12/27/https-ssl-certification.html)是安全的http协议，在使用的时候必须考虑SSL和数字证书的问题。

##0x00 获取HttpClient
###默认单例HttpClient
可以通过`new DefaultHttpClient()`获取默认HttpClient
``` java
/**
* 获取单例HttpClient
* @return
*/
public static HttpClient getHttpClient(String urlString){
    if(httpClient == null){
        httpClient = new DefaultHttpClient();
    }
    return httpClient;
}
```
###线程安全单例HttpClient
Android API中通过ClientConnectionManager接口可以获取线程安全的HttpClient，支持http和https。其中SSLSoketFactoryEX为自定义类继承了SSLSoketFactory，下面会介绍。
```java
/**
* 使用synchronized加锁，获取唯一HttpClient实例
* @return
*/
public static synchronized HttpClient getSaveHttpClient(){
    if(httpClient == null){
        try{
            KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
            trustStore.load(null, null);
            SSLSocketFactory ssf = new SSLSocketFactoryEx(trustStore);
            //允许所有主机的验证
             ssf.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);
            HttpParams params = new BasicHttpParams();
            //基本基本参数
            HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1);
            HttpProtocolParams.setContentCharset(params, CHARSET);
            HttpProtocolParams.setUseExpectContinue(params, true);
            //超时设置
            //从连接池中获取的超时时间
            ConnManagerParams.setTimeout(params, timeOut);
            //网络连接超时时间
            HttpConnectionParams.setConnectionTimeout(params, timeOut);
            //请求连接超时时间
            HttpConnectionParams.setSoTimeout(params, timeOut);
            //设置支持http、https协议
            SchemeRegistry schReg = new SchemeRegistry();
            schReg.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
            //使用自定义SSLSocketFactoryEx
            schReg.register(new Scheme("https", ssf, 443));
            //使用线程安全的连接管理来创建HttpClient
            ClientConnectionManager conManager = new ThreadSafeClientConnManager(params, schReg);
            httpClient = new DefaultHttpClient(conManager, params);
        }catch(Exception e){
            e.printStackTrace();
            return new DefaultHttpClient();
        }
    }
    return httpClient;
}
```
##0x01 实现get/post请求
###get请求
get请求主要是为了获取数据，当然也可以发送简单的数据，如URL:'http://hexo.com?name=XX&pwd=XX'
```java
/**
 * 利用get模式连接
 * @param urlStr
 * @return
 */
public static final String ConnectByGet(String urlStr){

    try{
        HttpGet httpGet = new HttpGet(checkUri(urlStr));
        HttpResponse response = getSaveHttpClient().execute(httpGet);
        try{
            if(response.getStatusLine().getStatusCode() == HttpStatus.SC_OK){
                InputStream inputStream = response.getEntity().getContent();
                InputStreamReader inputStreamReader = new InputStreamReader(inputStream);                            StringBuilder builder = new StringBuilder();
                char[] arrayOfChar = new char[64];
                int len;
                while((len = inputStreamReader.read(arrayOfChar)) != -1){
                    builder.append(arrayOfChar, 0, len);
                }
                return builder.toString();
            }
        }catch(Exception e){
            e.printStackTrace();
            LogUtils.d(TAG, "网络连接失败！");
        }finally{
            //保证所有实体内容已经被完全消耗
            response.getEntity().consumeContent();
        }
    }catch(Exception e){
        e.printStackTrace();
    }
    return null;
}
```

###post请求
post请求用来作为数据的发送接口。
```java
    /**
     * 利用post模式连接
     * @param urlStr 接口地址
     * @param params 参数（发送的数据）
     * @return
     */
    public static final String ConnectByPost(String urlStr, Map<String, String> params){
        try{
            HttpPost httpPost = new HttpPost(checkUri(urlStr));

            //添加上传数据
            List<BasicNameValuePair> paramsList = new ArrayList<BasicNameValuePair>();
            for(String key : params.keySet()){
                paramsList.add(new BasicNameValuePair(key, params.get(key)));
            }

            UrlEncodedFormEntity urlEntity = new UrlEncodedFormEntity(paramsList, CHARSET);
            httpPost.setEntity(urlEntity);
            HttpResponse response = getSaveHttpClient().execute(httpPost);

            try{
                //判断是否连接成功
                if(response.getStatusLine().getStatusCode() == HttpStatus.SC_OK){
                    //获取返回数据
                    InputStream inputStream = response.getEntity().getContent();
                    InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
                    StringBuilder builder = new StringBuilder();
                    char[] arrayOfChar = new char[64];
                    int len;
                    while((len = inputStreamReader.read(arrayOfChar)) != -1){
                        builder.append(arrayOfChar, 0, len);
                    }
                    return builder.toString();
                }
            }catch(Exception e){
                e.printStackTrace();
                LogUtils.d(TAG, "网络连接失败！");
            }finally{
                //保证所有实体内容已经被完全消耗
                response.getEntity().consumeContent();
            }
        }catch(Exception e){
            e.printStackTrace();
        }
        return null;
    }
```

##0x02 解决HTTPS证书问题
在Android使用https请求经常会遇到SSLPeerUnverifiedException异常：
<span style="color:red;">javax.net.ssl.SSLPeerUnverifiedException: No peer certificate</span>
这问题主要是因为https协议使用了SSL/TSL认证，认证过程中必须校验证书。
###信任全部证书
继承SSLSocketFactory，对里面方法进行重写。
```java
/**
 * 名称: HttpHelper.java
 * 描述: 证书管理
 *
 * @author a_xiang
 * @version v1.0
 * @created 2015年5月9日
 */
class SSLSocketFactoryEx extends SSLSocketFactory{

    SSLContext sslContext = SSLContext.getInstance("TLS");

    public SSLSocketFactoryEx(KeyStore truststore)
            throws NoSuchAlgorithmException, KeyManagementException,
            KeyStoreException, UnrecoverableKeyException {
        super(truststore);

        /** 证书管理*/
        TrustManager tm = new X509TrustManager(){

            @Override
            public void checkClientTrusted(X509Certificate[] chain,
                    String authType) throws CertificateException {
                // TODO Auto-generated method stub

            }

            @Override
            public void checkServerTrusted(X509Certificate[] chain,
                    String authType) throws CertificateException {
                // TODO Auto-generated method stub

            }

            @Override
            public X509Certificate[] getAcceptedIssuers() {
                // TODO Auto-generated method stub
                return null;
            }
        };
        sslContext.init(null, new TrustManager[] {tm}, null);
    }

    @Override
    public Socket createSocket(Socket socket, String host, int port,
            boolean autoClose) throws IOException, UnknownHostException {
        return sslContext.getSocketFactory().createSocket(socket, host, port, autoClose);
    }

    @Override
    public Socket createSocket() throws IOException {
        return sslContext.getSocketFactory().createSocket();
    }
}
```
###下载证书保存到本地
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
##0x03 附录
HttpHelper的完整代码

```java
/**
 * 名称: HttpHelper.java
 * 描述: Http工具类
 *
 * @author a_xiang
 * @version v1.0
 * @created 2015年3月27日
 */
public class HttpHelper {
    private static final String TAG = HttpHelper.class.getSimpleName(); 

    /** 超时时间 */
    private static final int timeOut = 10 * 1000;

    /** HttpClient */
    private static HttpClient httpClient = null;

    /** 编码格式*/
    private static final String CHARSET = HTTP.UTF_8;

    /**
     * 不能新建连接，只能通过对外的接口来获取HTTPClient实例
     */
    private HttpHelper(){    }

    /**
     * 获取单例HttpClient
     * @return
     */
    public static HttpClient getHttpClient(String urlString){
        if(httpClient == null){
            httpClient = new DefaultHttpClient();
        }
        return httpClient;
    }

    /**
     * 将字符串转化为URI
     * @param urlStr
     * @return
     */
    private static URI checkUri(String urlStr){
        try {
            URL localURL = new URL(urlStr);
            return localURL.toURI();
        } catch (MalformedURLException e) {
            e.printStackTrace();
            return null;
        } catch (URISyntaxException e) {
            e.printStackTrace();
            return null;
        }
    }

    /**
     * 使用synchronized加锁，获取唯一HttpClient实例
     * @return
     */
    public static synchronized HttpClient getSaveHttpClient(){
        if(httpClient == null){
            try{
                KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
                trustStore.load(null, null);
                SSLSocketFactory ssf = new SSLSocketFactoryEx(trustStore);
                //允许所有主机的验证
                ssf.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);
                
                HttpParams params = new BasicHttpParams();
                //基本基本参数
                HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1);
                HttpProtocolParams.setContentCharset(params, CHARSET);
                HttpProtocolParams.setUseExpectContinue(params, true);

                //超时设置
                //从连接池中获取的超时时间
                ConnManagerParams.setTimeout(params, timeOut);
                //网络连接超时时间
                HttpConnectionParams.setConnectionTimeout(params, timeOut);
                //请求连接超时时间
                HttpConnectionParams.setSoTimeout(params, timeOut);
                //设置支持http、https协议
                SchemeRegistry schReg = new SchemeRegistry();
                schReg.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
                //使用自定义SSLSocketFactoryEx
                schReg.register(new Scheme("https", ssf, 443));
                //使用线程安全的连接管理来创建HttpClient
                ClientConnectionManager conManager = new ThreadSafeClientConnManager(params, schReg);
                httpClient = new DefaultHttpClient(conManager, params);

            }catch(Exception e){
                e.printStackTrace();
                return new DefaultHttpClient();
            }
        }
        return httpClient;
    }

    /**
     * 利用get模式连接
     * @param urlStr
     * @return
     */
    public static final String ConnectByGet(String urlStr){

        try{
            HttpGet httpGet = new HttpGet(checkUri(urlStr));
            HttpResponse response = getSaveHttpClient().execute(httpGet);
            try{
                if(response.getStatusLine().getStatusCode() == HttpStatus.SC_OK){
                    InputStream inputStream = response.getEntity().getContent();
                    InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
                    StringBuilder builder = new StringBuilder();
                    char[] arrayOfChar = new char[64];
                    int len;
                    while((len = inputStreamReader.read(arrayOfChar)) != -1){
                        builder.append(arrayOfChar, 0, len);
                    }
                    return builder.toString();
                }
            }catch(Exception e){
                e.printStackTrace();
                LogUtils.d(TAG, "网络连接失败！");
            }finally{
                //保证所有实体内容已经被完全消耗
                response.getEntity().consumeContent();
            }
        }catch(Exception e){
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 利用post模式连接
     * @param urlStr
     * @param params
     * @return
     */
    public static final String ConnectByPost(String urlStr, Map<String, String> params){
        try{
            HttpPost httpPost = new HttpPost(checkUri(urlStr));

            //添加上传数据
            List<BasicNameValuePair> paramsList = new ArrayList<BasicNameValuePair>();
            for(String key : params.keySet()){
                paramsList.add(new BasicNameValuePair(key, params.get(key)));
            }

            UrlEncodedFormEntity urlEntity = new UrlEncodedFormEntity(paramsList, CHARSET);
            httpPost.setEntity(urlEntity);
            HttpResponse response = getSaveHttpClient().execute(httpPost);

            try{
                //判断是否连接成功
                if(response.getStatusLine().getStatusCode() == HttpStatus.SC_OK){
                    //获取返回数据
                    InputStream inputStream = response.getEntity().getContent();
                    InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
                    StringBuilder builder = new StringBuilder();
                    char[] arrayOfChar = new char[64];
                    int len;
                    while((len = inputStreamReader.read(arrayOfChar)) != -1){
                        builder.append(arrayOfChar, 0, len);
                    }
                    return builder.toString();
                }
            }catch(Exception e){
                e.printStackTrace();
                LogUtils.d(TAG, "网络连接失败！");
            }finally{
                //保证所有实体内容已经被完全消耗
                response.getEntity().consumeContent();
            }
        }catch(Exception e){
            e.printStackTrace();
        }
        return null;
    }

}

/**
 * 名称: HttpHelper.java
 * 描述: 证书管理
 *
 * @author a_xiang
 * @version v1.0
 * @created 2015年5月9日
 */
class SSLSocketFactoryEx extends SSLSocketFactory{

    SSLContext sslContext = SSLContext.getInstance("TLS");

    public SSLSocketFactoryEx(KeyStore truststore)
            throws NoSuchAlgorithmException, KeyManagementException,
            KeyStoreException, UnrecoverableKeyException {
        super(truststore);

        /** 证书管理*/
        TrustManager tm = new X509TrustManager(){

            @Override
            public void checkClientTrusted(X509Certificate[] chain,
                    String authType) throws CertificateException {
                // TODO Auto-generated method stub

            }

            @Override
            public void checkServerTrusted(X509Certificate[] chain,
                    String authType) throws CertificateException {
                // TODO Auto-generated method stub

            }

            @Override
            public X509Certificate[] getAcceptedIssuers() {
                // TODO Auto-generated method stub
                return null;
            }
        };
        sslContext.init(null, new TrustManager[] {tm}, null);
    }

    @Override
    public Socket createSocket(Socket socket, String host, int port,
            boolean autoClose) throws IOException, UnknownHostException {
        return sslContext.getSocketFactory().createSocket(socket, host, port, autoClose);
    }

    @Override
    public Socket createSocket() throws IOException {
        return sslContext.getSocketFactory().createSocket();
    }
}

```