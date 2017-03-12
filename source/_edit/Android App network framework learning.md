# Android客户端网络框架学习

by Maos

---

作为实习生，在研究公司客户端代码的时候发现里面用了一个网络框架，之前自己做的玩具代码没有真正用过这类框架，所以专门研究了一下。

在浏览项目时最常见的用法就是通过构建一个FormRequest发起一个请求，like this：

```java
FormRequest.Builder builder=new FormRequest.Builder();
builder.setMethod(Request.Method.POST);
builder.setUrl(requestHead + URL_NAME);
builder.addExtraData();
builder.addParam("param1", "aaa");
builder.addParam("param2", "bbb");
builder.addSuccessListener(new Response.Listener<String>() {
  @Override
  public void onResponse(String response) {
    if (!TextUtils.isEmpty(response)) {
      try {
        JSONObject jsonObject = new JSONObject(response);
        int status = jsonObject.optInt("status");
        // parse jsonObject...
      } catch (JSONException e) {
        Log.e("ResponseParseError", e.toString());
      }
    }
  }
});
builder.addFailureListener(new Response.ErrorListener() {
  @Override
  public void onErrorResponse(VolleyError error) {
    ToastCreater.showShortToast("Error！");
  }
});
OkVolley.getInstance().addRequest(builder.build());
```

使用起来非常的简洁舒适，先通过内部Builder初始化，然后加入到Volley的请求队列中去就行了。因此令我产生了一探究竟的想法，点进FormRequest去看，它的继承结构是这样：

![a](http://ol76jdosj.bkt.clouddn.com/a.jpg)

嗯。。查看源码可以发现它们逐级封装了URL、请求方式、Header、Body、优先级、Builder等等。将Http请求这个东西逐步具体化。

学习框架前这我突然想到一个问题，我们为什么要使用网络框架？（想起了在学校时老师让我们用socket写请求的时候。。。）

学过Android开发的人都知道，在处理网络模块的时候需要考虑很多问题，比如说：

​	1.Android主线程不能进行网络请求，需要另开线程处理

​	2.异步请求

​	3.缓存处理

​	4.网络问题处理，像二次连接、SSL的握手失败问题，以及从网络错误状态中恢复等	

​	5.blabla…

如果没有网络框架，面对这些问题，我们就必须一次又一次自己处理，效率低下不说，还容易导致bug满天飞。而且一个封装良好的网络框架可以提供很好的使用体验，就像上面那样。

首先回忆一下自己之前常用的基础Android网络API，自然就是它们两个：

​	**• HttpClient**：

​	功能丰富，但API结构复杂，已失去维护，在5.0版本后被谷歌弃用

​	**• HttpURLConnection**：

​	包很小，速度快，谷歌愿意进一步提高性能，2.3之前存在bug

HttpClient是apache的一个开源库，被Google加入了SDK里，应该是一个封装的比较完善的库，提供了非常丰富的API，Bug数量很少。但也正因为其API数量太多，结构复杂，很难在不破坏其兼容性的情况下对它进行扩展和升级，Google已经弃用了它。而HttpURLConnection则是一个轻量级的、快速的HTTP请求封装，在2.2之前存在一个著名的bug，就是连接池失效的问题。HttpURLConnection没有支持HttpClient那么多的功能，但是也正因为这样，Google认为提升它的性能和功能更加容易，因此推荐使用。这两个兄弟的故事告诉我们一个道理：一时的优势也可能变成劣势，劣势也能变成优势，祸兮福之所倚，福兮祸之所伏。。。（扯远了）

但是我们的项目并没有用这俩，而是使用了一个新的东东：okHttp。



## OkHttp

我的理解是OkHttp是一个现代、高效的类似HttpUrlConnection的东西。它具有很多优点，下面列出一些我比较能理解的：

1.支持SPDY黑科技，可以合并多个到同一个主机的请求

2.socket自动选择最好路线，并支持自动重连

3.拥有自动维护的socket连接池，减少握手次数

4.使用[Okio](https://goo.gl/F7rG1V)来大大简化数据的访问与存储

5.可以自动处理常见的网络问题

6.如果服务器配置了多个IP地址，当IP连接失败的时候，OkHttp会自动尝试下一个IP

7.同时支持同步或异步请求

8.从Android4.4开始HttpURLConnection的底层实现采用的是okHttp

### 引用Okhttp

Android Studio中可以在Gradle中进行如下配置：

```groovy
compile 'com.squareup.okio:okio:1.11.0'
compile 'com.squareup.okhttp3:okhttp:3.5.0'
```

### 探究OkHttp

OkHttp最核心的类应该就是一个OkHttpClient，它把okhttp包内封装的Request对象构造成Call对象。Call是HTTP请求的又一个封装，那为啥还要封装呢？我们看看OkHttpClient的newCall方法里做了什么。

```java
/**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
  	return new RealCall(this, request, false /* for web socket */);
  }
```

```java
RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
  this.client = client;
  this.originalRequest = originalRequest;
  this.forWebSocket = forWebSocket;
  this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
}
```

构造了一个Call的具体实现RealCall，这个东西拿到了OkHttpClient和原始的Request对象，然后做什么呢？我们打开Request的源码，可以看到它只封装了Head、Body、Method、Url、是否Https等基本的请求属性，而在RealCall里面则定义了execute、enqueue、cancel等方法和interceptors的处理。下面看看RealCall的这几个方法：

```java
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

enqueue就是定义了一个回调借口，封装一个AsyncCall，交给了Dispatcher去调度。AsyncCall是RealCall里的内部类，这是一个真正的线程，它里面有个execute方法，长这个样子：

```java
@Override protected void execute() {
  boolean signalledCallback = false;
  try {
    Response response = getResponseWithInterceptorChain();
    if (retryAndFollowUpInterceptor.isCanceled()) {
      signalledCallback = true;
      responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
    } else {
      signalledCallback = true;
      responseCallback.onResponse(RealCall.this, response);
    }
  } catch (IOException e) {
    if (signalledCallback) {
      // Do not signal the callback twice!
      Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
    } else {
      responseCallback.onFailure(RealCall.this, e);
    }
  } finally {
    client.dispatcher().finished(this);
  }
}
```

可以看到主要就是执行了 Response response = getResponseWithInterceptorChain()这一条，然后就是回调了。这个AsyncCall会被dispatcher发配到具体线程执行。

```java
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  try {
    client.dispatcher().executed(this);
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } finally {
    client.dispatcher().finished(this);
  }
}
```

这个execute和上一个enqueue中AsyncCall的execute很像，同样把工作交给了getResponseWithInterceptorChain去做。可以看到execute中显然是一个同步单线程执行的请求，它自己返回拿到的结果（OkHttp封装的Response）。可见enqueue和execute中都用了这个方法去真正执行Request，那么在getResponseWithInterceptorChain()中究竟做了什么呢？

```java
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));

  Interceptor.Chain chain = new RealInterceptorChain(
      interceptors, null, null, null, 0, originalRequest);
  return chain.proceed(originalRequest);
}
```

它把我们自己设置的interceptors，加上内置的interceptors，串成串。然后造了一个Chain对象，把拦截器和原始Request放进去，还预设了一个index为0，那这个东西有什么用呢？这个就要研究一下拦截器的执行过程了，在下节继续探究。经过上面的分析，OkHttp执行的一个大体过程就比较清晰了，如下图所示。

![okhttp](http://ol76jdosj.bkt.clouddn.com/okhttp.jpg)



### OkHttp拦截器

看RealInterceptorChain的构造器：

```java
public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
    HttpCodec httpCodec, Connection connection, int index, Request request) {
  this.interceptors = interceptors;
  this.connection = connection;
  this.streamAllocation = streamAllocation;
  this.httpCodec = httpCodec;
  this.index = index;
  this.request = request;
}
```

从它里面拿到的东西看它似乎要搞一个大事情。而且这里面connection、streamAllocation都是什么鬼？我们先不管它，继续看里面的代码，前面代码中在生成Chain之后就调用了它的proceed方法：

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
    Connection connection) throws IOException {
  if (index >= interceptors.size()) throw new AssertionError();

  calls++;

  // If we already have a stream, confirm that the incoming request will use it.
  if (this.httpCodec != null && !sameConnection(request.url())) {
    throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
        + " must retain the same host and port");
  }

  // If we already have a stream, confirm that this is the only call to chain.proceed().
  if (this.httpCodec != null && calls > 1) {
    throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
        + " must call proceed() exactly once");
  }

  // Call the next interceptor in the chain.
  RealInterceptorChain next = new RealInterceptorChain(
      interceptors, streamAllocation, httpCodec, connection, index + 1, request);
  Interceptor interceptor = interceptors.get(index);
  Response response = interceptor.intercept(next);

  // Confirm that the next interceptor made its required call to chain.proceed().
  if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
    throw new IllegalStateException("network interceptor " + interceptor
        + " must call proceed() exactly once");
  }

  // Confirm that the intercepted response isn't null.
  if (response == null) {
    throw new NullPointerException("interceptor " + interceptor + " returned null");
  }

  return response;
}
```

该方法中对httpCodec和请求url做了判断，然后开始执行第一个interceptor。这时重新构造了一个新的Chain对象，然后传给了interceptor，进入它的intercept方法。那么我们看看一个interceptor的实现长什么样：

```java
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();

    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```

这个自定义拦截器中有个重要的地方，必须通过chain.proceed(request)获取Response，也就是说它会再调用一次上面的方法，获取下一个interceptor，继续执行。为什么要这么做呢？因为拦截器里的工作是分拦截前和拦截后两部分进行的，如下图：

![interceptor](http://ol76jdosj.bkt.clouddn.com/interceptor2.jpg)

它以这样的方式分层运行，因此上面的代码可以保证我们一个拦截器不需要重写两个回调方法，而chain.proceed(request)将我们的intercept方法截成了两部分。因此，可以想到，在RealCall中最后加入的那个拦截器中，存在真正执行网络请求的代码，点进去看：

```java
/** This is the last interceptor in the chain. It makes a network call to the server. */
public final class CallServerInterceptor implements Interceptor {
  private final boolean forWebSocket;

  public CallServerInterceptor(boolean forWebSocket) {
    this.forWebSocket = forWebSocket;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    HttpCodec httpCodec = ((RealInterceptorChain) chain).httpStream();
    StreamAllocation streamAllocation = ((RealInterceptorChain) chain).streamAllocation();
    Request request = chain.request();

    long sentRequestMillis = System.currentTimeMillis();
    httpCodec.writeRequestHeaders(request);

    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
      request.body().writeTo(bufferedRequestBody);
      bufferedRequestBody.close();
    }

    httpCodec.finishRequest();

    Response response = httpCodec.readResponseHeaders()
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```

果然在这。这里面真正执行了请求，构建了Response。从它之后就开始逐级往回走，执行其他拦截器的拦截后操作。

拦截器的设计思想其实就是一种面向切面编程的体现，一个interceptor就是一个切面，而切入点就在OkHttpClient中。不同于Spring框架的动态代理实现方案，拦截器直接使用addInterceptor把切面切入到了切入点中，而在这个过程中OkHttpClient的源代码不需要有任何的变化。即通过将作用于某一些目标Request上的通用操作提取，实现了在不修改源码的前提下，在程序运行时动态地将代码作用于这一批Request上，使得我们的Request不需要预先写死很多也许不会用到的操作，真正实现了高效的可扩展性。



## OkVolley

**——“即使你要单独使用OkHttp，还是会再包一层的，这样就等价于Volley之流的框架，只是封装的好与坏而已。”**

我看到在我们的客户端中并没有直接使用OkHttp，因为OkHttp本质上还是HttpURLConnection那一级别的东西，它只处理了Http请求这一过程所需要解决的问题，而作为一个客户端网络框架来直接用的话还是不够方便的。而在我们的项目中，使用了Volley。

Volley 是 Google 推出的 Android 异步网络请求框架和图片加载框架。在 Google I/O 2013 大会上发布。

Volley的主要特点：

(1). **扩展性强**。Volley 中大多是基于接口的设计，可配置性强。

(2). Volley特别适合**数据量小，通信频繁**的网络操作。

(3).默认 Android2.3 及以上基于 HttpURLConnection，2.3 以下基于 HttpClient 实现。

(4). 提供简便的图片加载工具。

#### Volley的组成

**Volley：**

这是Volley 对开发者提供的主要API，我们在这里面创建请求队列。

**Request：**

各种具体Request继承自Request抽象基类。我们可以基于它创建具体的Request类，如ImageRequest、JsonRequest等。

**RequestQueue：**

我们创建的请求实例被加入到这个请求队列中，它里面包含1个CacheDispatcher和若干个（好像默认是4个）NetworkDispatcher，通过start方法启动它们。

**CacheDispatcher：**

这是一个线程，用于调度处理走缓存的请求。

**NetworkDispatcher：**

这也是一个线程，用于调度处理走网络的请求。

**ResponseDelivery：**

返回结果分发接口。

**HttpStack：**

这个类非常重要，在它的performRequest方法中真正处理 Http 请求，返回一个古老的HttpResponse请求结果。

**Network：**

调用HttpStack处理请求，将HttpStack返回的HttpResponse封装成可被ResponseDelivery处理的NetworkResponse。

**Cache：**

在sdcard中缓存请求结果。

所以整个Volley框架的执行流程应该就是下图这个样子：

![http://ol76jdosj.bkt.clouddn.com/volley.jpg](http://ol76jdosj.bkt.clouddn.com/volley.jpg)



### OkVolley的封装

最后最厉害的地方来了，我们前面介绍的OkHttp怎么和Volley结合呢？答案很简单，就是这张图：

![http://ol76jdosj.bkt.clouddn.com/okvolley.jpg](http://ol76jdosj.bkt.clouddn.com/okvolley.jpg)

只需要两步：自己重新创建一个继承HttpStack的OkHttpStack，在其PerformRequest中使用OkHttpClient进行请求执行与结果解析。然后在Volley中使用这个HttpStack创建RequestQueue，就可以使用了！

只需要简单的替换，就可以将Volley配置成基于OkHttp的OkVolley。这个封装充分体现了Volley的面向接口编程，而不是面向实现编程的设计原则，实现了强大的可扩展性。





