## OkHttp核心



### 3. StreamAllocation

Http 1.1虽然可以使用Keep-Alive建立长连接，但是发送的文本格式，并且是串行使用。

Http 2.0使用了多路复用技术。多个stream可以共用一个socket，每个tcp连接都是通过一个socket来完成的，socket对应一个host和port，如果有多个stream（即多个Request），都是连接在一个host和port上，所以它们可以共用一个socket，这样就可以减少每次连接都需要三次握手的时间。

OkHttp里面，负责连接的是RealConnection。

>  **StreamAllocation是用来连接Connections、Streams、Calls这个三个实体的**。

创建StreamAllocation，是在RetryAndFollowUpInterceptor的intercept方法中：

```java
@Override public Response intercept(Chain chain) throws IOException {
  Request request = chain.request();
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Call call = realChain.call();
  EventListener eventListener = realChain.eventListener();

  StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
      createAddress(request.url()), call, eventListener, callStackTrace);
  this.streamAllocation = streamAllocation;
```

然后通过拦截器链进行传递，直到ConnectInterceptor和CallServerInterceptor这两个拦截器中才用到。

```java
public final class ConnectInterceptor implements Interceptor {
    @Override public Response intercept(Chain chain) throws IOException {
        ...
        HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
        RealConnection connection = streamAllocation.connection();
        return realChain.proceed(request, streamAllocation, httpCodec, connection);
    }
}
public final class CallServerInterceptor implements Interceptor {
  	@Override public Response intercept(Chain chain) throws IOException {
    ...
        streamAllocation.noNewStreams();
    ...
  	}
}
```

StreamAllocation的构造函数：

```java
  public StreamAllocation(ConnectionPool connectionPool, Address address, Call call,
      EventListener eventListener, Object callStackTrace) {
    // OkHttpClient配置的连接池
    this.connectionPool = connectionPool;
    // 根据Request构建了 Address
    this.address = address;
    // RealCall对象
    this.call = call;
    this.eventListener = eventListener;
    this.routeSelector = new RouteSelector(address, routeDatabase(), call, eventListener);
    this.callStackTrace = callStackTrace;
  }
```

StreamAllocation的主要调用方法`newStream`和`noNewStream`。

* newStream方法主要找到一个可用的连接connection，并且返回一个对应http协议的编解码codec。

  ```java
    public HttpCodec newStream(
        OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    ... 
      try {
        RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
        HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);
  
        synchronized (connectionPool) {
          codec = resultCodec;
          return resultCodec;
        }
      } catch (IOException e) {
        throw new RouteException(e);
      }
    }
  ```

* noNewStream设置当前connection不再创建新的流

  ```java
    public void noNewStreams() {
      Socket socket;
      Connection releasedConnection;
      synchronized (connectionPool) {
        releasedConnection = connection;
        socket = deallocate(true, false, false);
        if (connection != null) releasedConnection = null;
      }
      closeQuietly(socket);
      if (releasedConnection != null) {
        eventListener.connectionReleased(call, releasedConnection);
      }
    }
  ```

  

