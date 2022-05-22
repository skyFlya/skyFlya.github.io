---
title: 源码学习 - OkHttp
tags:
  - 源码解析
categories: Android
updated: 1635686871000
date: 2021-10-31 21:28:18
---

> http的连接本质上是个socket，根据http协议，通过socket包装发送请求并获得返回结果

网路连接库一开始的样子如下代码所示，其实只要符合Http协议的请求，就可以和网络进行交互，类似于OkHttp的网络请求库，帮助开发者方便和屏蔽了Http协议中类似于请求头，重连、合并、代理、返回结果解析等等Http协议细节的应用层实现。
<!-- more -->
```java
val path = "http://www.baidu.com/"
        val host = "www.baidu.com"
        var socket: Socket? = null
        var streamWriter: OutputStreamWriter? = null
        var bufferedWriter: BufferedWriter? = null
        try {
            socket = Socket(host, 80)
            streamWriter = OutputStreamWriter(socket.getOutputStream())
            bufferedWriter = BufferedWriter(streamWriter)
            bufferedWriter.write("GET $path HTTP/1.1\r\n")
            bufferedWriter.write("Host: www.baidu.com\r\n")
            bufferedWriter.write("\r\n")
            bufferedWriter.flush()
            val myRequest = BufferedReader(InputStreamReader(socket.getInputStream(), "UTF-8"))
            var d = -1
            while (myRequest.read().also({ d = it }) != -1) {
                print(d.toChar())
            }
        } catch (e: IOException) {
            e.printStackTrace()
        }
```

## 整体流程

```java
public class GetExample {
  // 核网络管理者 - 核心类
  final OkHttpClient client = new OkHttpClient();

  String run(String url) throws IOException {
    // 请求搭建
    Request request = new Request.Builder().url(url).build();
    /*
      阻塞式execute -> 立即调用请求，并阻塞，直到响应可以处理或出现错误
      为了避免资源泄漏，调用者应该关闭Response，而Response又会关闭底层的ResponseBody。 //确保响应(和底层响应体)是关闭的
      注意，传输层的成功(接收HTTP响应代码、报头和正文)不一定表示应用层的成功:响应可能仍然表示不满意的HTTP响应代码，如404或500。
     */
    try (Response response = client.newCall(request).execute()) {
      return response.body().string();
    }
  }

  public static void main(String[] args) throws IOException {
    GetExample example = new GetExample();
    String response = example.run("https://raw.github.com/square/okhttp/master/README.md");
    System.out.println(response);
  }
}

```
![5hiy4O.png](https://z3.ax1x.com/2021/10/25/5hiy4O.png)

### 连接建立
Volley等很多网络请求框架很多底层都是通过 HTTPURLConnection 来与服务端建立连接的，而 OkHttp 就比较优秀了。因为 HTTP 协议是建立在 TCP/IP 协议基础之上的，底层还是走的 Socket，所以OkHttp 直接使用 Socket 来完成 HTTP 请求。

### OkHttpClient
官方推荐我们使用单例去创建OkHttpClient，重用所有HTTP调用的时候，性能是最佳的， 这是因为每个客户端都拥有自己的连接池和线程池。 重用连接和线程可减少延迟并节省内存。 相反，为每个请求创建一个客户端会浪费空闲池上的资源

### Request
采用Builder的方式进行设计，主要包含了url、method、headers、body和CacheControl组成的各种配置项

```java
    final Dispatcher dispatcher;  //分发器
    final Proxy proxy;  //代理
    final List<Protocol> protocols; //协议
    final List<ConnectionSpec> connectionSpecs; //传输层版本和连接协议
    final List<Interceptor> interceptors; //拦截器
    final List<Interceptor> networkInterceptors; //网络拦截器
    final ProxySelector proxySelector; //代理选择
    final CookieJar cookieJar; //cookie
    final Cache cache; //缓存
    final InternalCache internalCache;  //内部缓存
    final SocketFactory socketFactory;  //socket 工厂
    final SSLSocketFactory sslSocketFactory; //安全套接层socket 工厂，用于HTTPS
    final CertificateChainCleaner certificateChainCleaner; // 验证确认响应证书 适用 HTTPS 请求连接的主机名。
    final HostnameVerifier hostnameVerifier;    //  主机名字确认
    final CertificatePinner certificatePinner;  //  证书链
    final Authenticator proxyAuthenticator;     //代理身份验证
    final Authenticator authenticator;      // 本地身份验证
    final ConnectionPool connectionPool;    //连接池,复用连接
    final Dns dns;  //域名
    final boolean followSslRedirects;  //安全套接层重定向
    final boolean followRedirects;  //本地重定向
    final boolean retryOnConnectionFailure; //重试连接失败
    final int connectTimeout;    //连接超时
    final int readTimeout; //read 超时
    final int writeTimeout; //write 超时 
```

### Call
> 一个提供 HTTP 请求执行相关接口的接口类，具体的实现类是 RealCall
- 可以取消
- 此对象表示单个请求/响应对(流),因此不能执行两次

#### RealCall
>OkHttp的应用层和网络层之间的桥梁,包含了网络的连接、请求、响应和流处理整个流程,也是OkHttp中最关键核心的类

##### AsyncTimeout

此超时用在在后台线程执行超时时执行操作。 使用它来实现本地不支持的超时，例如写入时被阻止的套接字,子类应该覆盖timedOut以在发生超时时采取行动。 此方法将由共享看门狗线程调用，因此不应执行任何长时间运行的操作。 否则，我们可能会面临触发其他超时的风险。  
使用sink和source将此超时应用于流。 返回的值将超时应用于包装流上的每个操作。  
调用者应该在执行可能超时的工作之前调用enter ，然后退出。 exit的返回值表示是否触发了超时。 请注意，对timedOut的调用是异步的，可以在exit之后调用

```java
private val timeout = object : AsyncTimeout() {  
  override fun timedOut() {  
    // 取消请求  
  cancel()  
  }  
}.apply {  
  timeout(client.callTimeoutMillis.toLong(), MILLISECONDS)  
}
```

#####  execute
> 同步请求,  马上执行并阻塞直到可以处理响应或出现错误

```kotlin
  override fun execute(): Response {
    /**
     * Atomic就是原子性的意思，源码里用了Volatile属性，即能够保证在高并发的情况下只有一个线程能够访问这个属性值
     * executed 是一个原子变量，一般情况下，我们使用 AtomicBoolean 高效并发处理 “只初始化一次” 的功能要求,用 compareAndSet(false, true)
     * 判断多线程状态下，请求是否重新执行：如果值为false，则抛出一个IllegalStateException，并返回调用lazyMessage的结果。
     */
    check(executed.compareAndSet(false, true)) { "Already Executed" }
    // 超时计数开始
    timeout.enter()
    // 执行请求前处理 -> 栈跟踪，事件回调等等
    callStart()
    try {
      // 调度器开始执行
      client.dispatcher.executed(this)
      // 返回拦截器处理下的响应
      return getResponseWithInterceptorChain()
    } finally {
      // 调度器返回完成信号
      client.dispatcher.finished(this)
    }
  }
```

#####  enqueue
> 异步调度,  将请求放到队列中等到执行

RealCall.kt
```kotlin
override fun enqueue(responseCallback: Callback) {  
  // 首先判断当前请求是否已执行，如果已经执行则打印日志，并抛出 IllegalStateException 异常
  check(executed.compareAndSet(false, true)) { "Already Executed" }  
  callStart()  
  // 创建一个AsyncCall对象,放进分发器队列中
  client.dispatcher.enqueue(AsyncCall(responseCallback))  
}

private fun callStart() {
    // 返回一个对象，该对象包含在执行此方法时创建的堆栈跟踪
    this.callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()")
    eventListener.callStart(this)
}
```
Dispatcher.kt
```kotlin
  ///////////////////////////////////////////////////////////////////////////
  // https://zhuanlan.zhihu.com/p/261397170
  // ArrayDeque -  Java 集合中双端队列的数组实现
  // ArrayDeque 几乎没有容量限制，设计为线程不安全的，禁止 null 元素
  // ArrayDeque是Deque的实现类，可以作为栈来使用，效率高于Stack；也可以作为队列来使用，效率高于LinkedList。
  // ArrayDeque 大多数的额操作都在固定时间内运行，例外情况包括 remove，removeFirstOccurrence，removeLastOccurrence，contains，iterator.remove()，和批量操作，这些将以线性时间运行
  ///////////////////////////////////////////////////////////////////////////
  private val runningSyncCalls = ArrayDeque<RealCall>()

  internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      // 将请求 AsyncCall 添加到待执行队列
      readyAsyncCalls.add(call)
      // 判断当前请求是否已存在可复用的 hos
      if (!call.call.forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host)
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    // 将符合条件的Call从readyAsyncCalls 提升到runningAsyncCalls并在执行它们
    promoteAndExecute()
  }
```
RealCall.AsyncCall
```kotlin
    override fun run() {
      // 给当前AsyncCall异步线程设置名称
      threadName("OkHttp ${redactedUrl()}") {
        var signalledCallback = false
        timeout.enter()
        try {
          // 获取请求结果
          val response = getResponseWithInterceptorChain()
          signalledCallback = true
          // 触发相应回调
          responseCallback.onResponse(this@RealCall, response)
        } catch (e: IOException) {
          if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log("Callback failure for ${toLoggableString()}", Platform.INFO, e)
          } else {
            responseCallback.onFailure(this@RealCall, e)
          }
        } catch (t: Throwable) {
          cancel()
          if (!signalledCallback) {
            val canceledException = IOException("canceled due to $t")
            canceledException.addSuppressed(t)
            responseCallback.onFailure(this@RealCall, canceledException)
          }
          throw t
        } finally {
          // 关闭请求
          client.dispatcher.finished(this)
        }
      }
    }
  }
```

### Dispatcher(分发器)

> 用于管理其对应 OkHttpClient 的所有请求,对Call进行统一的控制，例如结束所有请求、获取线程池等等

-   readyAsyncCalls：一个新的异步请求首先会被加入该队列中
-   runningAsyncCalls：当前正在运行中的异步请求
-   runningSyncCalls：当前正在运行的同步请求

异步请求跟同步请求一样,最终都会调用到`getResponseWithInterceptorChain()`

### Interceptor(拦截器)


>Interceptor 接口作为一个拦截器的抽象概念，被设计为责任链上的单位节点，用于观察、拦截、处理请求等，例如添加 Header、重定向、数据处理等等。  
Interceptor 之间互相独立，每个 Interceptor 只负责自己关注的任务，不与其他 Interceptor 接触。

RealCall.kt
```kotlin
  @Throws(IOException::class)
  internal fun getResponseWithInterceptorChain(): Response {
    // TODO: 2021/8/13 okhttp的核心是拦截器，而拦截器所采用的设计模式是责任链设计，即每个拦截器只处理与自己相关的业务逻辑 https://zhuanlan.zhihu.com/p/340090732
    // 构建整个网络请求拦截
    val interceptors = mutableListOf<Interceptor>()
    // 添加Client的拦截器
    interceptors += client.interceptors
    // 添加失败重连的拦截器
    interceptors += RetryAndFollowUpInterceptor(client)
    // 添加请求桥梁拦截器 - 在对用户的请求头部加了一些信息，然后在获取到的响应中也做了一些处理。而这些处理对用户是透明的，减少了客户请求的工作
    interceptors += BridgeInterceptor(client.cookieJar)
    // 添加缓存拦截器 - 理来自缓存的请求并将响应写入缓存。
    interceptors += CacheInterceptor(client.cache)
    // 添加请求中拦截器 - 打开与目标服务器的连接并继续下一个拦截器。 网络可能用于返回的响应，或使用条件 GET 验证缓存的响应
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    // 这是链中的最后一个拦截器。 它对服务器进行网络调用,真正的网络请求从这里开始
    interceptors += CallServerInterceptor(forWebSocket)
    // 构建网络请求链 - 一个具体的拦截器链，承载着整个拦截器链：所有应用程序拦截器、OkHttp核心、所有网络拦截器，最后是网络调用者。
    val chain = RealInterceptorChain(
        call = this,
        interceptors = interceptors,
        index = 0,
        exchange = null,
        request = originalRequest,
        connectTimeoutMillis = client.connectTimeoutMillis,
        readTimeoutMillis = client.readTimeoutMillis,
        writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
      val response = chain.proceed(originalRequest)
      // 是否取消请求
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }
```
RealInterceptorChain.kt
```kotlin
  @Throws(IOException::class)
  override fun proceed(request: Request): Response {
    // 判断拦截器是否为空
    check(index < interceptors.size)
    // 请求数加1
    calls++
    if (exchange != null) {
      check(exchange.finder.sameHostAndPort(request.url)) {
        "network interceptor ${interceptors[index - 1]} must retain the same host and port"
      }
      check(calls == 1) {
        "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"
      }
    }
    // Call the next interceptor in the chain. 调用链中的下一个拦截器
    val next = copy(index = index + 1, request = request)
    // 获取当前的拦截器
    val interceptor = interceptors[index]
    // 开始一个个的执行每一个拦截器，每个拦截器的intercept都会调到 当前类的proceed ，直至最后一个CallServerInterceptor执行完
    @Suppress("USELESS_ELVIS")
    val response = interceptor.intercept(next) ?: throw NullPointerException("interceptor $interceptor returned null")
    if (exchange != null) {
      check(index + 1 >= interceptors.size || next.calls == 1) {
        "network interceptor $interceptor must call proceed() exactly once"
      }
    }
    check(response.body != null) { "interceptor $interceptor returned a response with no body" }
    return response
  }
```

#### Chain

> Interceptor 与 Chain 彼此互相依赖，互相调用，共同发展，形成了一个完美的调用链

Chain 被用来描述责任链，通过其中的 process 方法开始依次执行链上的每个节点，并返回处理后的 Response。  
Chain 的唯一实现为 RealInterceptorChain（下文简称 RIC），RIC 可以称之为**拦截器责任链**，其中的节点由 RealCall 中添加进来的 Interceptor 们组成。由于 Interceptor 的互相独立性，RIC 中还会包含一些公共参数及共享的对象。


![5hbpJx.png](https://z3.ax1x.com/2021/10/25/5hbpJx.png)

### About

[OkHttp源码深度解析](https://zhuanlan.zhihu.com/p/116777864)
[OkHttp源码解析](https://zhuanlan.zhihu.com/p/104813091)
[OkHttp源码原理](https://juejin.cn/post/7020027832977850381)