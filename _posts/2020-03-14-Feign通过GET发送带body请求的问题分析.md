---
layout:     post
title:      Feign通过GET发送带body请求的问题分析
subtitle:   微服务
date:       2020-03-14
author:     qin4zhang
header-img: img/post-bg-universe.jpg 
catalog: true 						# 是否归档
tags:								#标签
    - 微服务
    - Spring Cloud
    - Feign
---
# 注意
> 想法及时记录，实现可以待做。

## 问题描述
项目A通过Feign直接调用项目B的接口查询数据状态，项目B提供的接口是GET方法，可以带body进行请求的，原因是可以批量查询，避免参数过多。

插叙新的接口定义为：

```
@ApiOperation("批量获取邮件状态")
@GetMapping("status")
GenericResult<BatchMailStatusDTO> acquireMailStatus(@ApiParam("邮件id列表") @RequestBody List<Long> var1);
```

实际请求的时候，出现了异常，信息为：

```
com.t4f.gaea.exception.LogicException: Request method 'POST' not supported
    at com.t4f.gaea.spring.feign.LogicExceptionErrorDecoder.decode(LogicExceptionErrorDecoder.java:43) ~[gaea-spring-2.3.16.jar:na]
    at feign.SynchronousMethodHandler.executeAndDecode(SynchronousMethodHandler.java:143) ~[feign-core-9.7.0.jar:na]
    at feign.SynchronousMethodHandler.invoke(SynchronousMethodHandler.java:77) ~[feign-core-9.7.0.jar:na]
    at feign.ReflectiveFeign$FeignInvocationHandler.invoke(ReflectiveFeign.java:102) ~[feign-core-9.7.0.jar:na]
    at com.sun.proxy.$Proxy191.acquireMailStatus(Unknown Source) ~[na:na]
```

## 问题分析

GET接口带有body参数，虽说接口设计一般不建议这样使用，但是也没有明确禁止不能用。如果真的这样用有问题，那HTTP协议还不如直接禁止了，何必留下了这个方式呢？

还有个问题：接口定义的是GET请求，但是异常提示的却是POST请求，这明显是中间哪里出了问题。本着追根溯源的目的，准备好好研究一番，将Feign的debug日志打印出来，看看http的报文是啥样的。

注：打开Feign的日志debug级别如下：

```
Feign日志的配置
@Configuration
public class FeignConfig {

    @Bean
    Logger.Level feignLoggerLevel() {
        //这里记录所有，根据实际情况选择合适的日志level
        return Logger.Level.FULL;
    }
}

FeignClient配置日志
@FeignClient(value = "mail", configuration = FeignConfig.class)

申明日志级别
logging:
  level:
    com:
      xxx:
        MailClient: debug
```

经过异常配置，再次请求查看报文数据：

```
请求报文：
[GameMailClient#acquireMailStatus] ---> GET http://mail/status
[GameMailClient#acquireMailStatus] Content-Type: application/json
[GameMailClient#acquireMailStatus] Content-Length: 3
[GameMailClient#acquireMailStatus]
[GameMailClient#acquireMailStatus] [1]
[GameMailClient#acquireMailStatus] ---> END HTTP (3-byte body)
 
响应报文：
[GameMailClient#acquireMailStatus] <--- HTTP/1.1 500 (9421ms)
[GameMailClient#acquireMailStatus] connection: close
[GameMailClient#acquireMailStatus] content-type: application/json
[GameMailClient#acquireMailStatus] date: Fri, 15 Jan 2021 09:42
[GameMailClient#acquireMailStatus] source: ark-game-mail
[GameMailClient#acquireMailStatus] transfer-encoding: chunked
[GameMailClient#acquireMailStatus]
[GameMailClient#acquireMailStatus] {"success":false,"code":"500"}
[GameMailClient#acquireMailStatus] <--- END HTTP (78-byte body)
```

通过Feign打印的日志来看，Feign发出去的GET请求，那就继续跟进去看看最终发送出去的报文哪里有变化。

## 问题溯源
从异常堆栈信息中可以看到，`at feign.SynchronousMethodHandler.invoke(SynchronousMethodHandler.java:77) ~[feign-core-9.7.0.jar:na]`，就从 `SynchronousMethodHandler`这个类的77行开始看起。

```
71 @Override
72 public Object invoke(Object[] argv) throws Throwable {
73   RequestTemplate template = buildTemplateFromArgs.create(argv);
74   Retryer retryer = this.retryer.clone();
75   while (true) {
76     try {
77       return executeAndDecode(template);
78     } catch (RetryableException e) {
79       retryer.continueOrPropagate(e);
80       if (logLevel != Logger.Level.NONE) {
81         logger.logRetry(metadata.configKey(), logLevel);
82       }
83       continue;
84     }
85   }
86 }
```

根据77行的executeAndDecode方法

```
89 Request request = targetRequest(template);
90  
91 if (logLevel != Logger.Level.NONE) {
92   logger.logRequest(metadata.configKey(), logLevel, request);
93 }
94  
95 Response response;
96 long start = System.nanoTime();
97 try {
98   response = client.execute(request, options);
99   // ensure the request is set. TODO: remove in Feign 10
100   response.toBuilder().request(request).build();
101 } catch (IOException e) {
102   if (logLevel != Logger.Level.NONE) {
103     logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
104   }
105   throw errorExecuting(request, e);
106 }
```

从98行可以看到，有一个`client.execute`，说明是客户端在指定请求，但是client有多个实现，继续跟踪下去，最后发现调用的是feign client接口的默认实现

```
56 public static class Default implements Client {
57  
58   private final SSLSocketFactory sslContextFactory;
59   private final HostnameVerifier hostnameVerifier;
60  
61   /**
62    * Null parameters imply platform defaults.
63    */
64   public Default(SSLSocketFactory sslContextFactory, HostnameVerifier hostnameVerifier) {
65     this.sslContextFactory = sslContextFactory;
66     this.hostnameVerifier = hostnameVerifier;
67   }
68  
69   @Override
70   public Response execute(Request request, Options options) throws IOException {
71     HttpURLConnection connection = convertAndSend(request, options);
72     return convertResponse(connection).toBuilder().request(request).build();
73   }
```

着重看70行的execute方法，这是默认的执行方法。71行的convertAndSend重要如下：

```
76 final HttpURLConnection
77     connection =
78     (HttpURLConnection) new URL(request.url()).openConnection();
```

这个说明，默认的feign的客户端，其实就是jdk自带的HttpURLConnection做http的请求。

```
124 if (request.body() != null) {
125   if (contentLength != null) {
126     connection.setFixedLengthStreamingMode(contentLength);
127   } else {
128     connection.setChunkedStreamingMode(8196);
129   }
130   connection.setDoOutput(true);
131   OutputStream out = connection.getOutputStream();
132   if (gzipEncodedRequest) {
133     out = new GZIPOutputStream(out);
134   } else if (deflateEncodedRequest) {
135     out = new DeflaterOutputStream(out);
136   }
137   try {
138     out.write(request.body());
139   } finally {
140     try {
141       out.close();
142     } catch (IOException suppressed) { // NOPMD
143     }
144   }
145 }
```

如果请求带有body，就会将`connection.setDoOutput(true);`注意，这个很重要，请看`connection.getOutputStream()`的执行，这个最终会执行HttpURLConnection里面的getOutputStream0方法。

```
1319 private synchronized OutputStream getOutputStream0() throws IOException {
1320     try {
1321         if (!doOutput) {
1322             throw new ProtocolException("cannot write to a URLConnection"
1323                            + " if doOutput=false - call setDoOutput(true)");
1324         }
1325  
1326         if (method.equals("GET")) {
1327             method = "POST"; // Backward compatibility
1328         }
```

可以看到，doOutput如果不为true，这里就直接抛异常，说明feign是专门设置为true的，
但是，带有body的方法在jdk默认的请求中，会做个特殊的处理，遇到是GET请求，就会替换成POST请求，然后进行实际的http发送，原来如此！
这也就是为啥表面看起来，feign这一层全都是GET方法，最后的响应异常却提示POST不被支持，原来是jdk悄悄咪咪的做了处理。

## 解决方法
feign默认的http客户端走的是jdk的HttpURLConnection，而这个对于带有body的GET请求，会自动转为POST请求，这个不符合业务想要的结果。出现的问题就在于是GET带有body。

1. 切换http client的客户端，不要使用feign默认的客户端，也就是jdk自带的HttpURLConnection，可以使用apache的http client。
有关于http client的配置，可以看FeignAutoConfiguration，这个自动装配的配置类。
在pom文件添加即可指定客户端：

```
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
</dependency>
```

这个会自动从服父pom继承版本号。

2. GET方法不要带body了，就使用Query String的方式即可。

## 总结
1. GET方法带body，在RESTful API的设计中是不被建议的，建议用通用的接口设计规范会更好。
2. 实在要用这个，需要对底层使用的http client比较熟悉，能正常的传递消息，否则遇到这种全部都是默认的实现，就容易踩坑了。


