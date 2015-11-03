title: "Java中绕过HTTP代理"
date: 2015-11-03 09:49:27
categories: java
tags: [java, proxy]
toc: true
description: java中不使用HTTP代理，获取真实IP
---

####HttpConnection：
```java

 private static String postWithHttpUrlConnection(String urlStr, Map<String, String> params) {
 		HttpURLConnection httpURLConnection = null;
        try {
            InputStream inputStream = null;
            OutputStream outputStream = null;
            StringBuilder stringBuffer = new StringBuilder();

            URL url = new URL(urlStr);

            httpURLConnection = (HttpURLConnection) url.openConnection(Proxy.NO_PROXY);
            httpURLConnection.setDoInput(true);
            httpURLConnection.setDoOutput(true);
            httpURLConnection.setRequestMethod("POST");
            httpURLConnection.setConnectTimeout(3 * 1000);
            httpURLConnection.setReadTimeout(4 * 1000);
            httpURLConnection.setRequestProperty("Charset", "utf-8");


            outputStream = httpURLConnection.getOutputStream();
            outputStream.write(getParams(params));
            outputStream.flush();
            outputStream.close();

            httpURLConnection.connect();
            if (httpURLConnection.getResponseCode() == HttpURLConnection.HTTP_OK) {

                inputStream = httpURLConnection.getInputStream();
                BufferedReader buffer = new BufferedReader(new InputStreamReader(inputStream));

                String line;
                while (null != (line = buffer.readLine())) {
                    stringBuffer.append(line);
                }
                inputStream.close();
                return stringBuffer.toString();
            }
            return "failed";
        } catch (Exception e) {
            e.printStackTrace();
            return "failed";
        }
    } 
```


####HttpClient
```java
	 /**
     * 使用synchronized加锁，获取唯一HttpClient实例
     *
     * @return
     */
    public static synchronized HttpClient getSaveHttpClient() {
        if (httpClient == null) {
            try {
                HttpParams params = getHttpParams();

                //设置支持http、https协议
                SchemeRegistry schReg = new SchemeRegistry();
                schReg.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
                //使用默认证书校验
                schReg.register(new Scheme("https", SSLSocketFactory.getSocketFactory(), 443));

                //使用线程安全的连接管理来创建HttpClient
                ClientConnectionManager conManager = new ThreadSafeClientConnManager(params, schReg);
                httpClient = new DefaultHttpClient(conManager, params);
                // 不走代理
                HttpHost host = new HttpHost(ConnRouteParams.NO_HOST);
                httpClient.getParams().setParameter(ConnRouteParams.DEFAULT_PROXY, host);

            } catch (Exception e) {
                LogUtil.d(TAG, "ttpClient配置异常！");
                if (LogUtil.E) e.printStackTrace();
                return new DefaultHttpClient();
            }
        }
        return httpClient;
    }

```

