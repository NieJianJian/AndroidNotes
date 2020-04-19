## OkHttp源码分析

[基于OkHttp Version 3.12.0](https://square.github.io/okhttp/changelog_3x/#version-3120)

* 源码：

  [OkHttpClient](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/OkHttpClient.md)、[Dispatcher](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/Dispatcher.md)、[RealCall](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/RealCall.md)、[NamedRunnable](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/NamedRunnable.md)、[Interceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/Interceptor.md)、[RealInterceptorChain](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/RealInterceptorChain.md)、[RetryAndFollowUpInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/RetryAndFollowUpInterceptor.md)、[BridgeInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/BridgeInterceptor.md)、[CacheInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/CacheInterceptor.md)、[ConnectInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/ConnectInterceptor.md)、[CallServerInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/CallServerInterceptor.md)、[StreamAllocation](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/StreamAllocation.md)、[RealConnection](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/RealConnection.md)、[ConnectionPool](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/ConnectionPool.md)

* 目录

  * [1. OkHttp处理网络流程](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/OkHttp.md#1-okhttp处理网络流程)
  * [2. 拦截器分析](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/OkHttp.md#2-拦截器分析)
  * [3. OkHttp核心——StreamAllocation、ConnectionPool、RealConnection](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/OkHttp_core.md)

***

来看一个简单的异步GET请求：

```java
private void onClick() {
    Request.Builder builder = new Request.Builder().
            url("https://github.com/NieJianJian/AndroidNotes");
    builder.method("GET", null);
    final Request request = builder.build();
    OkHttpClient okHttpClient = new OkHttpClient();
    Call call = okHttpClient.newCall(request);
    call.enqueue(new Callback() {
        @Override
        public void onFailure(Call call, IOException e) {
            Log.i("niejianjian", " -> onFailure -> " + e.toString());
        }
        @Override
        public void onResponse(Call call, Response response) throws IOException {
            String res = response.body().string();
            Log.i("niejianjian", " -> res -> " + res);
        }
    });
}
```

**注意**：onResponse回调并非在UI线程。

如果想要调用同步GET请求，可以调用Call的execute方法。

### 1. OkHttp处理网络流程

* 请求处理分析

  调用OkHttpClient.newCall(request)进行execute和enqueue方法，当调用newCall方法时：	

  ```java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
  ```

  调用RealCall.newCall方法：

  ```java
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
  ```

  最后返回的是一个RealCall类型。RealCall实现了Call接口，所以我们调用Call的enqueue和execute方法，实际上调用的是RealCall对象中对应的方法。

* Call的执行方法

  同步请求，调用RealCall类中的execute方法，直接返回的是一个Response对象。

  ```java
  @Override public Response execute() throws IOException {
      synchronized (this) {
          if (executed) throw new IllegalStateException("Already Executed");
          executed = true;
      }
      captureCallStackTrace();
      timeout.enter();
      eventListener.callStart(this);
      try {
          client.dispatcher().executed(this);
          Response result = getResponseWithInterceptorChain();
          if (result == null) throw new IOException("Canceled");
          return result;
      } catch (IOException e) {
          e = timeoutExit(e);
          eventListener.callFailed(this, e);
          throw e;
      } finally {
          client.dispatcher().finished(this);
      }
  }
  ```

  异步请求，调用RealCall类中的enqueue方法

  ```java
  @Override public void enqueue(Callback responseCallback) {
      synchronized (this) {
          if (executed) throw new IllegalStateException("Already Executed");
          executed = true;
      }
      captureCallStackTrace();
      eventListener.callStart(this);
      client.dispatcher().enqueue(new RealCall.AsyncCall(responseCallback));
  }
  ```

  上面代码可以看到，最终请求都是有clinet.dispatcher返回的Dispatcher对象执行的。

* Dispatcher任务调度

  Dispatcher在OkHttpClient的构造函数中初始化的。

  Dispatcher主要控制并发的请求，主要变量如下：

  ```java
  // 最大并发请求数
  private int maxRequests = 64;
  // 每个主机的最大请求数
  private int maxRequestsPerHost = 5;
  // 消费者线程池
  private @Nullable ExecutorService executorService;
  // 准备执行的异步请求队列
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
  // 正在执行的异步请求队列
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();
  // 正在执行的同步请求队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  ```

  Dispatcher的构造方法如下：

  ```java
  public Dispatcher(ExecutorService executorService) {
      this.executorService = executorService;
  }
  public Dispatcher() { }
  public synchronized ExecutorService executorService() {
      if (executorService == null) {
          executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60,
                  TimeUnit.SECONDS, new SynchronousQueue<Runnable>(), 
                  Util.threadFactory("OkHttp Dispatcher", false));
      }
      return executorService;
  }
  ```

  上述代码可以看出，可以使用自己设定的线程池，如果没有，则会在请求网络前，创建默认的线程池。

  Diapatcher的execute方法执行如下，将任务添加到正在执行的同步请求队列。

  ```java
  synchronized void executed(RealCall call) {
      runningSyncCalls.add(call);
  }
  ```

  Dispatcher的enqueue方法执行如下：

  ```java
  void enqueue(RealCall.AsyncCall call) {
      synchronized (this) {
          readyAsyncCalls.add(call);
      }
      promoteAndExecute();
  }
  ```

  传递进来的是一个AsyncCall对象，该类是RealCall的内部类，在RealCall的enqueue方法

  ```java
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
  ```

  将回调封装成了AsyncCall对象。AsyncCall继承了NamedRunnable，AsyncCall构造函数如下：

  ```java
  AsyncCall(Callback responseCallback) {
     super("OkHttp %s", redactedUrl());
     this.responseCallback = responseCallback;
  }
  ```

  NamedRunnable类的构造函数中，只是将线程的的名字进行了封装。

  回到Diepatcher的enqueue方法中，调用了promoteAndExecute方法，代码如下：

  ```java
  private boolean promoteAndExecute() {
      assert (!Thread.holdsLock(this));
      List<RealCall.AsyncCall> executableCalls = new ArrayList<>();
      boolean isRunning;
      synchronized (this) {
          for (Iterator<RealCall.AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
              RealCall.AsyncCall asyncCall = i.next();
              if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
              if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.
              i.remove();
              executableCalls.add(asyncCall);
              runningAsyncCalls.add(asyncCall);
          }
          isRunning = runningCallsCount() > 0;
      }
      for (int i = 0, size = executableCalls.size(); i < size; i++) {
          RealCall.AsyncCall asyncCall = executableCalls.get(i);
          asyncCall.executeOn(executorService());
      }
      return isRunning;
  }
  ```

  该方法中主要的作用是，遍历将要运行的异步请求队列readyAsyncCalls，并且判断正在运行的队列是否已经超过了最大请求数，如果没有，就将队列中的元素移除，并添加到正在运行的请求的队列runningAsyncCalls中，然后调用AsyncCall的executeOn方法，执行该操作。

  接下来看AsyncCall的executeOn方法，代码如下：

  ```java
  void executeOn(ExecutorService executorService) {
      boolean success = false;
      try {
          executorService.execute(this); // 1
          success = true;
      } catch (RejectedExecutionException e) {
          ...
      } finally {
          if (!success) {
              client.dispatcher().finished(this); // 2
          }
      }
  }
  ```

  主要看代码注释1处，其实就是线程池去运行了线程。所以会调用到NamedRunnable的run方法中：

  ```java
  @Override public final void run() {
      String oldName = Thread.currentThread().getName();
      Thread.currentThread().setName(name);
      try {
          execute();
      } finally {
          Thread.currentThread().setName(oldName);
      }
  }
  ```

  设置线程的名字为处理过的名字，执行execute方法，然后将线程名字改回默认。

  来看AsyncCall的execute方法，代码如下：

  ```java
  @Override protected void execute() {
      boolean signalledCallback = false;
      timeout.enter();
      try {
          Response response = getResponseWithInterceptorChain(); // 1
          if (retryAndFollowUpInterceptor.isCanceled()) {
              signalledCallback = true;
              responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
          } else {
              signalledCallback = true;
              responseCallback.onResponse(RealCall.this, response); // 2
          }
      } catch (IOException e) {
          e = timeoutExit(e);
          if (signalledCallback) {
              // Do not signal the callback twice!
              Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
          } else {
              eventListener.callFailed(RealCall.this, e);
              responseCallback.onFailure(RealCall.this, e);
          }
      } finally {
          client.dispatcher().finished(this); // 3
      }
  }
  ```

  上面的代码注释1处，调用了getResponseWithInterceptorChain方法，返回到之前的代码，当我们调用Call.execute的同步请求代码时，里面也是通过该方法，返回了Response对象。说明，这个方法才是我们执行请求结果的地方。

  注释2处，就是我们的回调，处理onResponse和onFailure的原始调用处。

  我们先来看注释3，在之前我们调用AsyncCall的executeOn方法中，也有如下一行代码：

  ```java
  void executeOn(ExecutorService executorService) {
      boolean success = false;
      try {
          executorService.execute(this); // 1
          success = true;
      } catch (RejectedExecutionException e) {
          ...
      } finally {
          if (!success) {
              client.dispatcher().finished(this); // 2
          }
      }
  }
  ```

  AsyncCall的execute方法执行结束后，调用的Dispatcher的finished方法，AsyncCall的executeOn方法中真正执行线程之后，如果线程执行失败了，调用Dispatcher的finished方法，所以我们可以猜到，这个方法，是用来执行下一个队列中的任务的，我们来看代码：

  ```java
  void finished(AsyncCall call) {
     finished(runningAsyncCalls, call);
  }
  ```

  ```java
  private <T> void finished(Deque<T> calls, T call) {
      Runnable idleCallback;
      synchronized (this) {
          if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
          idleCallback = this.idleCallback;
      }
      boolean isRunning = promoteAndExecute(); // 1
      if (!isRunning && idleCallback != null) {
          idleCallback.run();
      }
  }
  ```

  代码中我们可以看到，又调用回到了promoteAndExecute方法中。所以不管是任务执行失败，还是任务执行结束，都回到这里，继续执行下一条任务。

* Interceptor拦截器

  我们回到RealCall下请求网络的方法getResponseWithInterceptorChain中，代码如下：

  ```java
  Response getResponseWithInterceptorChain() throws IOException {
      List<Interceptor> interceptors = new ArrayList<>();
      // 添加所有自定义的拦截器
      interceptors.addAll(client.interceptors());
      interceptors.add(retryAndFollowUpInterceptor);
      interceptors.add(new BridgeInterceptor(client.cookieJar()));
      interceptors.add(new CacheInterceptor(client.internalCache()));
      interceptors.add(new ConnectInterceptor(client));
      if (!forWebSocket) {
          interceptors.addAll(client.networkInterceptors());
      }
      interceptors.add(new CallServerInterceptor(forWebSocket));
      Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
              originalRequest, this, eventListener, client.connectTimeoutMillis(),
              client.readTimeoutMillis(), client.writeTimeoutMillis());
      return chain.proceed(originalRequest);
  }
  ```

  该方法所做的就是构成了一个拦截器的链，然后通过依次执行每一个不同功能的拦截器来获取服务器的响应返回。主要添加的拦截器有：

  * 用户自定义的拦截器，注意要实现Interceptor.Chain接口，并且要取调用下一个拦截器。可以在请求服务器之前做一些自己的逻辑处理。
  * RetryAndFollowUpInterceptor：失败之后自动重链和重定向。
  * BridgeInterceptor：校接和适配拦截器，主要补充用户创建请求当中的一些请求头Content-Type
  * CacheInterceptor：缓存。读取缓存直接返回和更新缓存。
  * ConnectInterceptor：与服务器创建链接，创建可以用的RealConnection(对java.io和java.nio进行了封装)
  * CallServerInterceptor：向服务器发送请求和接收数据。将请求写入IO流，再从IO流中读取响应数据

  我们看到，上述代码中，将拦截器集合和原始的request封装到了拦截器链中，先来看一下Interceptor类接口，代码如下：

  ```java
  public interface Interceptor {
      Response intercept(okhttp3.Interceptor.Chain chain) throws IOException;
      interface Chain {
          Request request();
          Response proceed(Request request) throws IOException;
          @Nullable
          Connection connection();
          Call call();
          int connectTimeoutMillis();
          okhttp3.Interceptor.Chain withConnectTimeout(int timeout, TimeUnit unit);
          int readTimeoutMillis();
          okhttp3.Interceptor.Chain withReadTimeout(int timeout, TimeUnit unit);
          int writeTimeoutMillis();
          okhttp3.Interceptor.Chain withWriteTimeout(int timeout, TimeUnit unit);
      }
  }
  ```

  真实的对象是RealInterceptorChain，它是一个具体的拦截器链，包含整个拦截器链：所有应用程序拦截器，OkHttp核心，所有网络拦截器，最后是网络调用者。我们来看它的proceed方法：

  ```java
  public Response proceed(Request request, StreamAllocation streamAllocation,
                          HttpCodec httpCodec,
                          RealConnection connection) throws IOException {
      ...
      // Call the next interceptor in the chain.
      RealInterceptorChain next = new RealInterceptorChain(interceptors, 
              streamAllocation, httpCodec, connection, index + 1, 
              request, call, eventListener, connectTimeout, readTimeout, writeTimeout);
      Interceptor interceptor = interceptors.get(index);
      Response response = interceptor.intercept(next);
      ...
      return response;
  }
  ```

  上面的代码中，RealInterceptorChain的构造函数中传入的一个参数为index + 1，这说明，如果下一次访问，就调用下一个拦截器。

  通过得到拦截器在调用自身的intercept方法传入刚刚创建的拦截器链next，这样每次在执行的拦截器内部调用拦截器的时候，都是调用的下一个拦截器，因为index + 1，这样就把所有的拦截器链构成了一个完整的链条。

  每次执行拦截器的时候，都会先调用到RealInterceptorChain的proceed方法，进入去执行单个拦截器的intercept方法。这样一层一层的传进去，再一层一层的返回Response结果，通过拦截链的功能，就在请求前对请求进行了拦截加工，返回结果后，也对结果进行了拦截加工，最后返回真正的结果实体。

   从执行RetryAndFollowUpInterceptor－>执行BridgeInterceptor－>执行CacheInterceptor－>执行ConnectInterceptor－>执行CallServerInterceptor－>依次将响应一一返回－>返回到ConnectInterceptor－>返回到CacheInterceptor－>返回到BridgeInterceptor－>返回到RetryAndFollowUpInterceptor。

***

### 2. 拦截器分析

* RetryAndFollowUpInterceptor：失败之后自动重链和重定向。
* BridgeInterceptor：校接和适配拦截器，主要补充用户创建请求当中的一些请求头Content-Type
* CacheInterceptor：缓存。读取缓存直接返回和更新缓存。
* ConnectInterceptor：与服务器创建链接，创建可以用的RealConnection(对java.io和java.nio进行了封装)
* CallServerInterceptor：向服务器发送请求和接收数据。将请求写入IO流，再从IO流中读取响应数据

主要用到了以上五个拦截器：

#### 2.1 RetryAndFollowUpInterceptor

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;
    // 最终计数，用来记录重连接的次数。
    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
        if (canceled) {
            streamAllocation.release();
            throw new IOException("Canceled");
        }
        Response response;
        boolean releaseConnection = true;
        try {
            response = realChain.proceed(request, streamAllocation, null, null);
            releaseConnection = false;
        } catch (RouteException e) {
            if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
                throw e.getFirstConnectException();
            }
            releaseConnection = false;
            continue;
        } catch (IOException e) {
            // recover方法判断一些特殊情况，如果是一些无法恢复的连接，则返回false，如禁止重连、
            // 请求体无法再次发送以及一些致命的异常，此时就会直接抛出异常，如果无关紧要，返回true
            boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
            if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
            releaseConnection = false;
            continue;
        } finally {
            // 抛出了未经检查的异常，释放资源
            if (releaseConnection) {
                streamAllocation.streamFailed(null);
                streamAllocation.release();
            }
        }
        // 上一次的response若存在，则将上一次的response的body置为null，赋值到response
        if (priorResponse != null) {
            response = response.newBuilder()
                    .priorResponse(priorResponse.newBuilder()
                            .body(null)
                            .build())
                    .build();
        }
        Request followUp;
        try {
            // 根据响应头和响应码判断是否需要重连，如果不需要则返回null
            followUp = followUpRequest(response, streamAllocation.route());
        } catch (IOException e) {
            streamAllocation.release();
            throw e;
        }
        // 没有request返回，释放资源，返回结果
        if (followUp == null) {
            streamAllocation.release();
            return response;
        }
        closeQuietly(response.body());
        // 如果到了此处，说明没有return response，意味着需要重连接
        // 判断重定向记录的次数，有没有超过20次，超过就不再重连接
        if (++followUpCount > MAX_FOLLOW_UPS) {
            streamAllocation.release();
            throw new ProtocolException("Too many follow-up requests: " + followUpCount);
        }
        if (followUp.body() instanceof UnrepeatableRequestBody) {
            streamAllocation.release();
            throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
        }
        // 检查如果不是相同的连接，则重连
        if (!sameConnection(response, followUp.url())) {
            streamAllocation.release();
            streamAllocation = new StreamAllocation(client.connectionPool(),
                    createAddress(followUp.url()), call, eventListener, callStackTrace);
            this.streamAllocation = streamAllocation;
        } else if (streamAllocation.codec() != null) {
            throw new IllegalStateException("Closing the body of " + response
                    + " didn't close its backing stream. Bad interceptor?");
        }
        // 重新设置requst，并把当前的response赋值给到priorResponse，继续while循环
        request = followUp;
        priorResponse = response;
    }
} 
```

* 设置计数器，记录重连次数，最多20次
* 发生异常判断异常是否是致命的，如果是，则抛出异常不再执行嗲吗
* 判断是否是未经过检查的，如果是，释放资源
* 上一次请求返回的response还在，就将body置为null
* 根据响应体和响应码判断是否需要重连，如果不需要，直接返回response。
* 判断重连次数，超过20次不再执行。
* 检查是否是相同的连接，不是则继续重连
* 重新设置request，并把当前的response赋值给到priorResponse，继续while循环。

#### 2.2 BridgeInterceptor

```java
@Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();
    RequestBody body = userRequest.body();
    // 设置request的header
    if (body != null) {
        MediaType contentType = body.contentType();
        if (contentType != null) {
            requestBuilder.header("Content-Type", contentType.toString());
        }
        ...
    }
    ... // 省略一些header设置
    // 加载cookie
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
        requestBuilder.header("Cookie", cookieHeader(cookies));
    }
    // 调用next拦截器。
    Response networkResponse = chain.proceed(requestBuilder.build());
    // 如果cookie不为空，将信息保存到cookie中
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
    //对拦截器返回的response进行处理。如果支持gzip压缩，则有Okio处理
    Response.Builder responseBuilder = networkResponse.newBuilder().request(userRequest);
    if (transparentGzip
            && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
            && HttpHeaders.hasBody(networkResponse)) {
        GzipSource responseBody = new GzipSource(networkResponse.body().source());
        Headers strippedHeaders = networkResponse.headers().newBuilder()
                .removeAll("Content-Encoding")
                .removeAll("Content-Length")
                .build();
        responseBuilder.headers(strippedHeaders);
        String contentType = networkResponse.header("Content-Type");
        responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }
    return responseBuilder.build();
}
```

* 主要设置了header的Content-Type、Content-Length、Transfer-Encoding、Host、Connection为Keep-Alive、Accept-Encoding、Cookie、User-Agent
* 判断是否需要加载cookie
* 调用下一个拦截器
* 如果cookie不为空，将信息保存到cookie
* 判断networkResponse是否支持gzip压缩，如果支持则通过Okio进行压缩

cookieJar是通过构造函数传入的：

```java
public BridgeInterceptor(CookieJar cookieJar) {
   this.cookieJar = cookieJar;
}
```

在RealCall创建BridgeInterceptor的时候，从OkHttpClient获取的，默认设置没有cookie。

```java
cookieJar = CookieJar.NO_COOKIES;
```

#### 2.3 CacheInterceptor 

其构造函数传入一个IternalCache对象，由OkHttoClient提供。

```java
interceptors.add(new CacheInterceptor(client.internalCache()));
```

OkHttpClient提供了设置缓存的方法，默认为null，

```java
void setInternalCache(@Nullable InternalCache internalCache) {
    this.internalCache = internalCache;
    this.cache = null;
}
public Builder cache(@Nullable Cache cache) {
    this.cache = cache;
    this.internalCache = null;
    return this;
}
```

internalCache和cache是相斥的，一个有值，另一个则为空。从上述源码可以看出，cache方法是public修饰，说明如果创建缓存策略，只能通过Builder创建cache，不能创建internalCache。

Cache代码如下：

```java
public final class Cache implements Closeable, Flushable {
    public Cache(File directory, long maxSize) {
        this(directory, maxSize, FileSystem.SYSTEM);
    }
    final InternalCache internalCache = new InternalCache() {
    ...
```

Cache中有一个InternalCache对象，并实现了其相关方法，同时采用了DiskLruCache进行缓存，缓存的内容就是response，通过url地址进行了md5加密作为key来进行增删改查。

同时注意下，在构建Cache的时候，必须要传入要保存的文件路径和最大缓存的大小。

* CacheStrategy

  就是给定一个request和cached response，根据里面设置的条件决定去使用网络请求还是缓存的response。从创建的Factory中可以看到返回的CacheStrategy就是根据不同的条件来将CacheStrategy中的networkRequest和cacheResponse进行设置值。

CacheInterceptor的具体逻辑：

```java
@Override public Response intercept(Chain chain) throws IOException {
    // 根据请求url判断有没有存储着缓存
    Response cacheCandidate = cache != null
            ? cache.get(chain.request()) : null;
    long now = System.currentTimeMillis();
    // 将request和缓存的response传入到缓存策略中，获取缓存策略中的networkRequest和cacheResponse
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    ...
    // 不请求网络并且缓存不存在或已经过期，返回504错误
    if (networkRequest == null && cacheResponse == null) {
        return new Response.Builder()
                .request(chain.request())
                .protocol(Protocol.HTTP_1_1)
                .code(504)
                .message("Unsatisfiable Request (only-if-cached)")
                .body(Util.EMPTY_RESPONSE)
                .sentRequestAtMillis(-1L)
                .receivedResponseAtMillis(System.currentTimeMillis())
                .build();
    }
    // 不进行网络请求，并且缓存可以用，返回缓存的response
    if (networkRequest == null) {
        return cacheResponse.newBuilder()
                .cacheResponse(stripBody(cacheResponse))
                .build();
    }
    // 缓存无效，直接请求服务器，进入到下一个拦截器返回对应的response
    Response networkResponse = null;
    try {
        networkResponse = chain.proceed(networkRequest);
    } finally {
        ...
    }
    // 如果缓存有设置
    if (cacheResponse != null) {
        // 请求到的code为304，说明服务器没有的结果没有更新，直接返回缓存并更新缓存。
        if (networkResponse.code() == HTTP_NOT_MODIFIED) {
            Response response = cacheResponse.newBuilder()
                    .headers(combine(cacheResponse.headers(), networkResponse.headers()))
                    .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
                    .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
                    .cacheResponse(stripBody(cacheResponse))
                    .networkResponse(stripBody(networkResponse))
                    .build();
            networkResponse.body().close();
            // Update the cache after combining headers but before stripping the
            // Content-Encoding header (as performed by initContentStream()).
            cache.trackConditionalCacheHit();
            cache.update(cacheResponse, response);
            return response;
        } else {
            closeQuietly(cacheResponse.body());
        }
    }
    // 使用网络请求
    Response response = networkResponse.newBuilder()
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
    // 根据情况进行缓存操作
    if (cache != null) {
        // 有数据，并且允许缓存，则将数据进行缓存
        if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
            // Offer this request to the cache.
            CacheRequest cacheRequest = cache.put(response);
            return cacheWritingResponse(cacheRequest, response);
        }
        if (HttpMethod.invalidatesCache(networkRequest.method())) {
            try {
                cache.remove(networkRequest);
            } catch (IOException ignored) {
                // The cache cannot be written.
            }
        }
    }
    return response;
}
```

* 加载本地缓存，以及创建缓存策略进行判断，由于没有设置，缓存默认是null。
* 将request和缓存的response传入到缓存策略中，获取缓存策略中的networkRequest和cacheResponse，然后根据返回的值来返回不同的response，具体有以下几种情况：
* 不请求网络，缓存也为null，则直接返回504
* 不请求网络，缓存不为null，直接返回缓存
* 请求网络，直接调用下一个拦截器去请求数据。
* 缓存有效，则和服务器返回的response进行对比，如果code为304，说明服务器未更新，缓存可用，直接返回缓存数据，并且更新缓存。
* 如果有缓存策略，将服务器的response缓存到本地。

#### 2.4 ConnectInterceptor

与服务器建立连接

```java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    //获取request和StreamAllocation
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    //获取StreamAllocation的流
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    //获取的streamAllocation连接
    RealConnection connection = streamAllocation.connection();
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

#### 2.5 CallServerInterceptor

真正的向服务器发送请求，并且得到服务器返回的数据.

```java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();
    long sentRequestMillis = System.currentTimeMillis();
    realChain.eventListener().requestHeadersStart(realChain.call());
    // 写入请求头
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);
    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
        // 如果发送到请求中含有100-continue(客户端在发送request之前，需要先判断服务器是否愿意
        // 接受客户端发来的消息主体)，则只有服务器返回100-continue的应答之后，才会把数据发送给服务器
        if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
            httpCodec.flushRequest();
            //开始请求服务器，检查返回的response中是否含有100-continue
            realChain.eventListener().responseHeadersStart(realChain.call());
            responseBuilder = httpCodec.readResponseHeaders(true);
        }
        // 写入请求数据
        if (responseBuilder == null) {
            // Write the request body if the "Expect: 100-continue" expectation was met.
            realChain.eventListener().requestBodyStart(realChain.call());
            long contentLength = request.body().contentLength();
            CountingSink requestBodyOut =
                    new CountingSink(httpCodec.createRequestBody(request, contentLength));
            BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
            request.body().writeTo(bufferedRequestBody);
            bufferedRequestBody.close();
            realChain.eventListener()
                    .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
        } else if (!connection.isMultiplexed()) {
            streamAllocation.noNewStreams();
        }
    }
    //完成网络请求，实际上就是将信息写入到socket的输出流中
    httpCodec.finishRequest();
    // 读取响应头
    if (responseBuilder == null) {
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(false);
    }
    Response response = responseBuilder
            .request(request)
            .handshake(streamAllocation.connection().handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();
    int code = response.code();
    ...
    if (forWebSocket && code == 101) {
        ...
    } else {
        // 读取服务器返回的内容
        response = response.newBuilder()
                .body(httpCodec.openResponseBody(response))
                .build();
    }
    ...
    return response;
}
```

***

### 3.  OkHttp核心

[ OkHttp核心——StreamAllocation、ConnectionPool、RealConnection](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/OkHttp_core.md)



***

### 参考链接

[OkHttp之getResponseWithInterceptorChain（一）](https://blog.csdn.net/nihaomabmt/article/details/88187205)

[OkHttp之getResponseWithInterceptorChain（二）](https://blog.csdn.net/nihaomabmt/article/details/88416148)

