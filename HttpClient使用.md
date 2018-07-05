# HttpClient使用

1、HttpClient的简介

​     HttpClient 是 Apache Jakarta Common 下的子项目，可以用来提供高效的、最新的、功能丰富的支持 HTTP 协议的客户端编程工具包，并且它支持 HTTP 协议最新的版本和建议。

2，HttpClient使用步骤

 1. 创建HttpClient对象。

    ```
    HttpClient httpClient = HttpClients. createDefault();
    ```

    HttpClient的实现类有：

    ![https://github.com/zayebuza/QA-java/blob/master/md_image/httpClient.png]()

 2. 创建请求方法的实例，并指定请求URL。如果需要发送GET请求，创建HttpGet对象；如果需要发送POST请求，创建HttpPost对象。

 3. 如果需要发送请求参数，可调用HttpGet、HttpPost共同的setParams(HetpParams params)方法来添加请求参数；对于HttpPost对象而言，也可调用setEntity(HttpEntity entity)方法来设置请求参数。

    Get

```java
  HttpClient httpClient = HttpClients. createDefault();
  HttpGet httpGet = new HttpGet("Https://www.baidu.com");
  CloseableHttpResponse response = ((CloseableHttpClient) httpClient).execute(httpGet);
  Assert.assertEquals(200,response.getStatusLine().getStatusCode());
  response.close();//最后一定要关闭
```

 	Put   

```java
HttpClient httpClient = HttpClients. createDefault();
HttpPost httpPost = new HttpPost("http://XXXXXXX.com");
List<NameValuePair> param = new ArrayList<>();
param.add(new BasicNameValuePair("userName","admin"));
param.add(new BasicNameValuePair("password","123456"));
UrlEncodedFormEntity entry = new UrlEncodedFormEntity(param,"utf-8");
httpPost.setEntity(entry);
CloseableHttpResponse response = ((CloseableHttpClient) httpClient).execute(httpPost);
Assert.assertEquals(200,response.getStatusLine().getStatusCode());
response.close(); //最后一定要关闭
```

先通过一个NameValuePair集合来存放待提交的参数,并将这个参数集合传入到一个UrlEncodedFormEntity中,然后调用HttpPost的setEntity()方法将构建好的UrlEncodedFormEntity传入: 

	4. 调用HttpClient对象的execute(HttpUriRequest request)发送请求，该方法返回一个HttpResponse。
	5. 调用HttpResponse的getAllHeaders()、getHeaders(String name)等方法可获取服务器的响应头；调用HttpResponse的getEntity()方法可获取HttpEntity对象，该对象包装了服务器的响应内容。程序可通过该对象获取服务器的响应内容。
	6. 释放连接。无论执行方法是否成功，都必须释放连接



参考：	https://blog.csdn.net/wangpeng047/article/details/19624529/