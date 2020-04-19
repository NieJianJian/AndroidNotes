## OkHttp核心——StreamAllocation、RealConnection、ConnectionPool

### 1. StreamAllocation

Http 1.1虽然可以使用Keep-Alive建立长连接，但是发送的文本格式，并且是串行使用。

Http 2.0使用了多路复用技术。多个stream可以共用一个socket，每个tcp连接都是通过一个socket来完成的，socket对应一个host和port，如果有多个stream（即多个Request），都是连接在一个host和port上，所以它们可以共用一个socket，这样就可以减少每次连接都需要三次握手的时间。

OkHttp里面，负责连接的是RealConnection。

>  **StreamAllocation是用来连接Connections、Streams、Calls这个三个实体的**。

**HTTP通信** 执行 **网络请求**`Call`  需要在 **连接**`Connection` 上建立一个新的 **流**`Stream`，我们将 `StreamAllocation` 称之 **流** 的桥梁，它负责为一次 **请求** 寻找 **连接** 并建立 **流**，从而完成远程通信。

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

* 接下来看newStream获取可用连接的流程

  ```java
    private RealConnection findHealthyConnection(...) throws IOException {
      while (true) {
        RealConnection candidate = findConnection(...);
        // 如果是一个新创建的连接，则直接返回
        synchronized (connectionPool) {
          if (candidate.successCount == 0) {
            return candidate;
          }
        }
        // 如果不是新连接，检查这个连接是不是已经准备承载新的流了（socket连接有没有问题）
        // 如果检查有问题，返回false，则继续执行while循环，否则就直接返回新创建的连接了。
        if (!candidate.isHealthy(doExtensiveHealthChecks)) {
          noNewStreams();
          continue;
        }
        return candidate;
      }
    }
  ```

* 接下来查看具体的查找连接过程findConnection方法：

  返回一个用于执行新stream的connection，优先选择现有的connection，其次是connection pool中的，最后是创建新的流。

  ```java
    private RealConnection findConnection(...) throws IOException {
      boolean foundPooledConnection = false;
      RealConnection result = null;
      Route selectedRoute = null;
      Connection releasedConnection;
      Socket toClose;
      synchronized (connectionPool) {
        ...
        // 尝试使用当前的connection，小心该connection是不是已经禁止分配新的流。
        releasedConnection = this.connection;
        toClose = releaseIfNoNewStreams();
        if (this.connection != null) {
          result = this.connection;
          releasedConnection = null;
        }
        if (!reportedAcquired) {
          // If the connection was never reported acquired, don't report it as released!
          releasedConnection = null;
        }
  
        if (result == null) {
          // 尝试从pool中进行获取connection，如果找到赋值给this.connection。
          Internal.instance.get(connectionPool, address, this, null);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
          } else {
            selectedRoute = route;
          }
        }
      }
      // 如果获取的现有connection已经禁止分配新流，则调用socket.close
      closeQuietly(toClose);
  		...
      if (result != null) {
        return result;
      }
      //  查看是否需要选择其他路由，这是一个阻塞操作。
      boolean newRouteSelection = false;
      if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
        newRouteSelection = true;
        routeSelection = routeSelector.next();
      }
      synchronized (connectionPool) {
        if (newRouteSelection) {
          // 如果有路由，则去连接池中查找对应的连接
          List<Route> routes = routeSelection.getAll();
          for (int i = 0, size = routes.size(); i < size; i++) {
            Route route = routes.get(i);
            Internal.instance.get(connectionPool, address, this, route);
            if (connection != null) {
              foundPooledConnection = true;
              result = connection;
              this.route = route;
              break;
            }
          }
        }
        if (!foundPooledConnection) {
          ...
          // 如果连接池中没有找到对应的连接，就创建一个RealConnection
          result = new RealConnection(connectionPool, selectedRoute);
          acquire(result, false);
        }
      }
      // 如果我们在连接池中找到的话，就直接返回connection
      if (foundPooledConnection) {
        eventListener.connectionAcquired(call, result);
        return result;
      }
      // 新创建的连接，开始执行TCP + TLS 握手连接，这是个阻塞操作。
      result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
          connectionRetryEnabled, call, eventListener);
      routeDatabase().connected(result.route());
      Socket socket = null;
      synchronized (connectionPool) {
        reportedAcquired = true;
        // 将connection添加到pool中
        Internal.instance.put(connectionPool, result);
        // If another multiplexed connection to the same address was created concurrently, 
        // then release this connection and acquire that one.
        if (result.isMultiplexed()) {
          socket = Internal.instance.deduplicate(connectionPool, address, this);
          result = connection;
        }
      }
      closeQuietly(socket);
      eventListener.connectionAcquired(call, result);
      return result;
    }
  ```

  * 尝试使用当前connection，注意当前connection是否已经禁止分配新的流。
  * 如果当前connection获取的为null，则从ConnectionPool获取，如果获取到，返回该连接。
  * 如果还是没有获取到，查看是否需要选择其他路由
  * 如果需要选择，就再次从Pool中查询相关连接
  * 如果找到相关连接，就返回。如果没有找到，就new RealConnection。
  * 如果是新创建的连接，就执行connect方法，开始TCP+TSL握手操作，并将该连接添加到pool。

* acquire方法，创建新的RealConnection之后调用，该方法需要和release方法成对出现。

  ```java
    public void acquire(RealConnection connection, boolean reportedAcquired) {
      assert (Thread.holdsLock(connectionPool));
      if (this.connection != null) throw new IllegalStateException();
  
      this.connection = connection;
      this.reportedAcquired = reportedAcquired;
      connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
    }
  ```

* release方法

  ```java
    public void release() {
      Socket socket;
      Connection releasedConnection;
      synchronized (connectionPool) {
        releasedConnection = connection;
        socket = deallocate(false, true, false);
        if (connection != null) releasedConnection = null;
      }
      closeQuietly(socket);
      if (releasedConnection != null) {
        Internal.instance.timeoutExit(call, null);
        eventListener.connectionReleased(call, releasedConnection);
        eventListener.callEnd(call);
      }
    }
  ```

  其中的deallocate方法中，调用到了release(RealConnection )方法

  ```java
    private void release(RealConnection connection) {
      for (int i = 0, size = connection.allocations.size(); i < size; i++) {
        Reference<StreamAllocation> reference = connection.allocations.get(i);
        if (reference.get() == this) {
          connection.allocations.remove(i);
          return;
        }
      }
      throw new IllegalStateException();
    }
  ```

  可以看到，acquire和release方法中，都一个相关的操作

  ```java
  connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  connection.allocations.remove(i);
  ```

  RealConnection的allocations对象是：

  ```java
  /** Current streams carried by this connection. */
  public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();
  ```

***

### 2. RealConnection

RealConnection是Connection的实现类，代表着链接socket的链路，如果拥有了一个RealConnection就代表了我们已经跟服务器有了一条通信链路。建立链路，说明在这个类实现了三次握手。

在StreamAllocation的findConnection方法中，如果新创建的RealConnection，最终调用了如下代码：

```java
result.connect(connectTimeout, readTimeout, writeTimeout, ...);
```

result就是RealConnection类型的。所以，从这一刻开始建立连接。

我们来看RealConnection的一些成员变量：

```java
private static final int MAX_TUNNEL_ATTEMPTS = 21;
private final ConnectionPool connectionPool;
private final Route route;
// 下面这些字段，由connect方法初始化，并且绝对不会再次赋值。
private Socket rawSocket; // 底层socket
private Socket socket; // 应用层socket
private Handshake handshake; // 握手
private Protocol protocol; // 协议
private Http2Connection http2Connection; // htpp2的连接
private BufferedSource source; // 与服务器交互的输入流
private BufferedSink sink; // 与服务器交互的输出流
// 下面的字段用于追踪连接状态，并有ConnectionPool管理
public boolean noNewStreams; // 为true，此链接上不能再创建新的流。
public int successCount; // 成功连接的次数，主要用于判断是不是新创建的链接。
// 此链接可以承载的最大的流的个数
public int allocationLimit = 1;
// 此链接当前承载的流
public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();
```

allocations是关联StreamAllocation，用来记录当前connection上建立了哪些流，通过StreamAllocation的acquire和release方法来进行集合元素的添加和移除。

**网络连接三次握手的方法connect**：

```java
public void connect(int connectTimeout, int readTimeout, int writeTimeout,
                    int pingIntervalMillis, boolean connectionRetryEnabled, Call call,
                    EventListener eventListener) {
  	// 连接是否已经建立，如果是，抛出异常
    if (protocol != null) throw new IllegalStateException("already connected");
    RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);
    // 判断是不是https协议，sslSocketFactory返回null则不是
    if (route.address().sslSocketFactory() == null) {
        if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
            throw new RouteException(new UnknownServiceException(
                    "CLEARTEXT communication not enabled for client"));
        }
        String host = route.address().url().host();
        if (!Platform.get().isCleartextTrafficPermitted(host)) {
            throw new RouteException(new UnknownServiceException(
                    "CLEARTEXT communication to " + host + " not permitted by network security policy"));
        }
    } else {
        if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
            throw new RouteException(new UnknownServiceException(
                    "H2_PRIOR_KNOWLEDGE cannot be used with HTTPS"));
        }
    }
  	// 链接开始
    while (true) {
        try {
            // 如果要求隧道模式，建立通道连接，通常不是这种
            if (route.requiresTunnel()) {
                connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
                if (rawSocket == null) {
                    break;
                }
            } else {
                // 一般走这条逻辑
                connectSocket(connectTimeout, readTimeout, call, eventListener);
            }
            // 建立协议
            establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
            eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
            break;
        } catch (IOException e) {
            closeQuietly(socket);
            closeQuietly(rawSocket);
            socket = null;
            rawSocket = null;
            source = null;
            sink = null;
            handshake = null;
            protocol = null;
            http2Connection = null;
            eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);
            if (routeException == null) {
                routeException = new RouteException(e);
            } else {
                routeException.addConnectException(e);
            }
            if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
                throw routeException;
            }
        }
    }
    if (route.requiresTunnel() && rawSocket == null) {
        ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
                + MAX_TUNNEL_ATTEMPTS);
        throw new RouteException(exception);
    }
    if (http2Connection != null) {
        synchronized (connectionPool) { 
            allocationLimit = http2Connection.maxConcurrentStreams();
        }
    }
}
```

* 判断连接是否已经建立，如果是，抛出异常，否则继续创建步骤。
* 基于是不是https连接的基础上，在进行一些安全性的判断。比如是否包含明文、当前设备是否允许明文传输等。
* 开始建立连接，如果满足使用HTTP通道代理HTTPS，就建立隧道连接，否则建立普通连接。
* 通过调用establishProtocol建立协议
* 如果是HTTP/2，则设置相关属性。

首先看第一个判断，这个判断是返回Https的配置是否为null。

```java
if (route.address().sslSocketFactory() == null)
```

判断的是Address的sslSocketFactory字段，返回的是ssl套接字工厂，如果不是HTTPS，就返回null

```java
public @Nullable SSLSocketFactory sslSocketFactory() { return sslSocketFactory;}
```

sslSocketFactory字段的初始化是在Address的构造函数，创建Address的构造函数是在RetryAndFollowUpInterceptor的intercept方法中会调用createAddress方法：

```java
  private Address createAddress(HttpUrl url) {
    SSLSocketFactory sslSocketFactory = null;
    HostnameVerifier hostnameVerifier = null;
    CertificatePinner certificatePinner = null;
    if (url.isHttps()) {
      sslSocketFactory = client.sslSocketFactory();
      hostnameVerifier = client.hostnameVerifier();
      certificatePinner = client.certificatePinner();
    }
    return new Address(url.host(), url.port(), client.dns(), client.socketFactory(),
        sslSocketFactory, hostnameVerifier, certificatePinner, client.proxyAuthenticator(),
        client.proxy(), client.protocols(), client.connectionSpecs(), client.proxySelector());
  }
```

会判断url是不是https类型的，如果是，就将client的相关参数赋值。所以不是https的类型的链接，上述判断null是满足的。Client中对sslSocketFactory默认赋值，最终会调用到AndroidPlatform平台下的方法，下面的方法返回一个TLS协议的SSLContext对象，最后在封装成SSLSocketFactory对象返回。就完成了赋值。

```java
SSLContext.getInstance("TLSv1.2");
```

> 拓展：SSL是HTTPS下的一个加密协议，是为了解决HTTP下传输是明文的问题，SSL是基于HTTP之下，TCP之上的一种协议，所以HTTPS = HTTP + SSL。TLS是是SSL的升级版协议。

接下来开始连接的时候，会判断：

```java
if (route.requiresTunnel()){
```

```java
// 如果此路由通过HTTP隧道，代理HTTPS，则返回true
public boolean requiresTunnel() {
   return address.sslSocketFactory != null && proxy.type() == Proxy.Type.HTTP;
}
```

sslSocketFactory的值上面说过，如果不是https，就为null。proxy.type()是在哪里赋值的呢？

Proxy的type是一个枚举类型，Typ表示代理类型：

```java
public enum Type {
    // 表示直接连接或没有代理
    DIRECT,
    // 代表高级协议（例如HTTP或FTP）的代理。
    HTTP,
    // 代表SOCKS（V4或V5）代理。
    SOCKS
};
```

Proxy有两个构造函数，其中一个如下：如果没有指定proxy，会创建默认的代理，并将type赋值为DIRECT。

```java
public final static Proxy NO_PROXY = new Proxy();
private Proxy() {
    type = Type.DIRECT;
}
```

proxy是在OkHttpClient的Builder中进行指定和初始化的，如果没有指定，都是默认的。

综上所述，我们普通请求使用的，都是connectSocket方法来创建链接。

```java
  private void connectSocket(int connectTimeout, int readTimeout, Call call,
      EventListener eventListener) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();
    // 根据代理类型来选择socket类型，是代理还是直连
    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);
    eventListener.connectStart(call, route.socketAddress(), proxy);
    rawSocket.setSoTimeout(readTimeout);
    try {
      // 连接socket，android平台调用AndroidPlatform的方法
      // 实际调用的是socket.connect(address, connectTimeout);
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }
    try {
      // 得到输入流、输出流
      source = Okio.buffer(Okio.source(rawSocket));
      sink = Okio.buffer(Okio.sink(rawSocket));
    } catch (NullPointerException npe) {
      if (NPE_THROW_WITH_NULL.equals(npe.getMessage())) {
        throw new IOException(npe);
      }
    }
  }
```

* 根据代理类型创建socket，3种情况创建普通socket，分别是直连（无代理）、明文的HTTP、SOCKS代理。

  * 非SOCKS代理的情况下，调用SocketFactory创建
  * SOCKS代理的情况下，直接new一个Socket，将proxy传递进去做参数。

* 为Socket设置超时时间10s

* 根据平台类型调用实际平台的方法，Android调用AndroidPlatform下的方法，里面实际是调用

  ```java
  socket.connect(address, connectTimeout);
  ```

* 用Okio来创建输入流和输出流。

设置了SOCKS代理的情况下，仅有的特别之处在于，是通过传入proxy手动创建Socket。route的socketAddress包含目标HTTP服务器的域名。由此可见SOCKS协议的处理，主要是在Java标准库的java.net.Socket中处理，对于外界而言，就好像是HTTP服务器直接建立连接一样，因此连接时传入的地址都是HTTP服务器的域名。

而对于明文的HTTP代理的情况下，这里没有任何特殊处理。route的socketAddress包含着代理服务器的IP地址。HTTP代理自身会根据请求及相应的实际内容，建立与HTTP服务器的TCP连接，并转发数据。

不管是建立隧道链接，还是建立普通链接，最后都需要建立协议，回到RealConnection的connect方法中， 在川里连接后，有如下一行代码：

```java
establishProtocol(connectionSpecSelector, pingIntervalMillis, call, eventListener);
```

```java
  private void establishProtocol(ConnectionSpecSelector connectionSpecSelector,
      int pingIntervalMillis, Call call, EventListener eventListener) throws IOException {
    // 如果不是SSL
    if (route.address().sslSocketFactory() == null) {
      // 判断是不是http2
      if (route.address().protocols().contains(Protocol.H2_PRIOR_KNOWLEDGE)) {
        socket = rawSocket;
        protocol = Protocol.H2_PRIOR_KNOWLEDGE;
        startHttp2(pingIntervalMillis);
        return;
      }
      socket = rawSocket;
      protocol = Protocol.HTTP_1_1;
      return;
    }
    // 如果是SSL
    eventListener.secureConnectStart(call);
    // 创建TLS连接
    connectTls(connectionSpecSelector);
    eventListener.secureConnectEnd(call, handshake);
    // 判断是不是http2
    if (protocol == Protocol.HTTP_2) {
      startHttp2(pingIntervalMillis);
    }
  }

  private void startHttp2(int pingIntervalMillis) throws IOException {
    socket.setSoTimeout(0); // HTTP/2 connection timeouts are set per-stream.
    http2Connection = new Http2Connection.Builder(true)
        .socket(socket, route.address().url().host(), source, sink)
        .listener(this)
        .pingIntervalMillis(pingIntervalMillis)
        .build();
    http2Connection.start();
  }
```

根据是否是https和是否是http2来做不同的操作，如果是http2，则初始化Http2Connection对象。

* 对于明文传输，则设置socket和protocol，对于加密传输，则创建Tls连接

TLS链接的connectTls方法：

```java
  private void connectTls(ConnectionSpecSelector connectionSpecSelector) throws IOException {
    Address address = route.address();
    SSLSocketFactory sslSocketFactory = address.sslSocketFactory();
    boolean success = false;
    SSLSocket sslSocket = null;
    try {
      // Create the wrapper over the connected socket.
      sslSocket = (SSLSocket) sslSocketFactory.createSocket(
          rawSocket, address.url().host(), address.url().port(), true /* autoClose */);

      // Configure the socket's ciphers, TLS versions, and extensions.
      ConnectionSpec connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket);
      if (connectionSpec.supportsTlsExtensions()) {
        Platform.get().configureTlsExtensions(
            sslSocket, address.url().host(), address.protocols());
      }

      // Force handshake. This can throw!
      sslSocket.startHandshake();
      // block for session establishment
      SSLSession sslSocketSession = sslSocket.getSession();
      Handshake unverifiedHandshake = Handshake.get(sslSocketSession);

      // Verify that the socket's certificates are acceptable for the target host.
      if (!address.hostnameVerifier().verify(address.url().host(), sslSocketSession)) {
        X509Certificate cert = (X509Certificate) unverifiedHandshake.peerCertificates().get(0);
        throw new SSLPeerUnverifiedException("Hostname " + address.url().host() + " not verified:"
            + "\n    certificate: " + CertificatePinner.pin(cert)
            + "\n    DN: " + cert.getSubjectDN().getName()
            + "\n    subjectAltNames: " + OkHostnameVerifier.allSubjectAltNames(cert));
      }

      // Check that the certificate pinner is satisfied by the certificates presented.
      address.certificatePinner().check(address.url().host(),
          unverifiedHandshake.peerCertificates());

      // Success! Save the handshake and the ALPN protocol.
      String maybeProtocol = connectionSpec.supportsTlsExtensions()
          ? Platform.get().getSelectedProtocol(sslSocket)
          : null;
      socket = sslSocket;
      source = Okio.buffer(Okio.source(socket));
      sink = Okio.buffer(Okio.sink(socket));
      handshake = unverifiedHandshake;
      protocol = maybeProtocol != null
          ? Protocol.get(maybeProtocol)
          : Protocol.HTTP_1_1;
      success = true;
    } catch (AssertionError e) {
      if (Util.isAndroidGetsocknameError(e)) throw new IOException(e);
      throw e;
    } finally {
      if (sslSocket != null) {
        Platform.get().afterHandshake(sslSocket);
      }
      if (!success) {
        closeQuietly(sslSocket);
      }
    }
  }
```

TLS连接是对原始TCP连接的一个封装，以及听过TLS握手，及数据手法过程中的加解密等功能。在Java中，用SSLSocket来描述。上面建立的TLS连接的过程大体为：

1、用SSLSocketFactory基于原始的TCP Socket，创建一个SSLSocket。

2、并配置SSLSocket。

3、在前面选择的ConnectionSpec支持TLS扩展参数时，配置TLS扩展参数。

4、启动TLS握手

5、TLS握手完成之后，获取证书信息。

6、对TLS握手过程中传回来的证书进行验证。

7、在前面选择的ConnectionSpec支持TLS扩展参数时，获取TLS握手过程中顺便完成的协议协商过程所选择的协议。这个过程主要用于HTTP/2的ALPN扩展。

8、OkHttp主要使用Okio来做IO操作，这里会基于前面获取到SSLSocket创建于执行的IO的BufferedSource和

BufferedSink等，并保存握手信息以及所选择的协议。

现在回到最开始的地方，StreamAllocation方中调用findHealthyConnection方法去寻找一个可用的连接，最后找到或者创建后，返回一个RealConnection实例

```java
RealConnection resultConnection = findHealthyConnection(connectTimeout, ...);
HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);
```

上述方法调用了RealConnection的newCodec方法，返回了一个HttpCodec实例。

```java
  public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
      StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {
      return new Http2Codec(client, chain, streamAllocation, http2Connection);
    } else {
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }
```

该方法中，根据Http2Connection是否为null，来决定创建Http1Codec还是Http2Codec的实例。

接下来看其中的一个方法isEligible(Address, Route)方法，这个方法主要是判断面对给出的addres和route，这个realConnetion是否可以重用。

```java
public boolean isEligible(Address address, @Nullable Route route) {
    // 如果连接创建流达到上限或者不能再创建新的流，那么返回false。
    if (allocations.size() >= allocationLimit || noNewStreams) return false;
    // 如果非host域不一样，那么返回false
    if (!Internal.instance.equalsNonHost(this.route.address(), address)) return false;
    // 如果host域完全匹配，则可以重用
    if (address.url().host().equals(this.route().address().url().host())) {
      return true; // This connection is a perfect match.
    }
    // 如果host不匹配，http2情况下也可以重用。
    if (http2Connection == null) return false;
    // 2. The routes must share an IP address. This requires us to have a DNS address for both
    // hosts, which only happens after route planning. We can't coalesce connections that use a
    // proxy, since proxies don't tell us the origin server's IP address.
    if (route == null) return false;
    if (route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (this.route.proxy().type() != Proxy.Type.DIRECT) return false;
    if (!this.route.socketAddress().equals(route.socketAddress())) return false;
    // 3. This connection's server certificate's must cover the new host.
    if (route.address().hostnameVerifier() != OkHostnameVerifier.INSTANCE) return false;
    if (!supportsUrl(address.url())) return false;
    // 4. Certificate pinning must match the host.
    try {
      address.certificatePinner().check(address.url().host(), handshake().peerCertificates());
    } catch (SSLPeerUnverifiedException e) {
      return false;
    }
    return true; // The caller's address can be carried by this connection.
  }
```

***

### 3. ConnectionPool

ConnectionPool用来管理http和http/2的链接，以便减少网络请求延迟。同一个address将共享同一个connection。该类实现了复用连接的目标。

来看一下该类的字段：

```java
// 后台用于清除过期链接的线程池，每个线程池最多只运行一个线程，线程池执行程序允许对池本身进行回收。
private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */, 
        Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));
// 每个Address的最大空闲连接数。
private final int maxIdleConnections;
// socket的KeepAlive时间
private final long keepAliveDurationNs;
private final Runnable cleanupRunnable = new Runnable() {
        @Override public void run() { 
        while (true) { 
            long waitNanos = cleanup(System.nanoTime());
            if (waitNanos == -1) return;
            if (waitNanos > 0) {
                long waitMillis = waitNanos / 1000000L;
                waitNanos -= (waitMillis * 1000000L);
                synchronized (ConnectionPool.this) {
                    try {
                        ConnectionPool.this.wait(waitMillis, (int) waitNanos);
                    } catch (InterruptedException ignored) {
                    }
                }
            }
        } 
    }
};
// 连接的双向队列。
private final Deque<RealConnection> connections = new ArrayDeque<>();
// 失败的路由黑名单数据库。
final RouteDatabase routeDatabase = new RouteDatabase();
boolean cleanupRunning; // 清理任务正在执行的的标识。
```

* executor线程池，类似于CacheThreadPool，里面用的是没有容量的SynchronousDueue。
* Deque，双向队列，具有队列和栈的性质，经常在缓存中被使用，里面维护的RealConnection，也就是socket物理连接的包装。

ConnectionPool的构造方法如下：

```java
public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
}
public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    this.maxIdleConnections = maxIdleConnections;
    this.keepAliveDurationNs = timeUnit.toNanos(keepAliveDuration);

    // Put a floor on the keep alive duration, otherwise cleanup will spin loop.
    if (keepAliveDuration <= 0) {
      throw new IllegalArgumentException("keepAliveDuration <= 0: " + keepAliveDuration);
    }
}
```

* 默认空闲的socket最大连接数是5，keepalive时间为5分钟。
* maxIdleConnections是值每个地址上最大的空闲连接数。所以OkHttp只是限制与同一个远程服务器的空闲连接数量，对整体的空闲连接并没有限制。

构造函数实例化是在OkHttpClient的Builder实例话的时候，调用的无参构造函数，所以一个OkHttpClient只有一个ConnectionPool。

ConnectionPool各个方法的调用并没有直接对外暴露，是通过OkHttpClient的Internal接口统一调用的。

ConnectionPool提供了对Deque< RealConnection>进行操作的方法，如put、get等，分别对对应的是放入连接、获取连接。

```java
  void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```

```java
  @Nullable RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
```

在put方法之前，需要先清理空闲的线程。在get方法中，会遍历connections列表，判断连接是否合格。

自动会收连接是根据StreamAllocation引用计数是否为0来实现自动会收连接的。put时调用。

cleanupRunnnable线程中，不断的调用cleanUp方法来进行清理，并返回下次需要清理的间隔时间，然后调用wait方法进行等待以释放锁与时间片。等待时间到了，再次进行清理，不断的循环下去。

```java
long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;
    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
        for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
            RealConnection connection = i.next();
            // 如果该连接正在被使用，继续搜索。	
            if (pruneAndGetAllocationCount(connection, now) > 0) {
                inUseConnectionCount++;
            }
            continue;
        }
        idleConnectionCount++;
        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
            longestIdleDurationNs = idleDurationNs;
            longestIdleConnection = connection;
        }
    } 
    if (longestIdleDurationNs >= this.keepAliveDurationNs
            || idleConnectionCount > this.maxIdleConnections) { // 2
        connections.remove(longestIdleConnection);
    } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
    } else if (inUseConnectionCount > 0) {
        return keepAliveDurationNs;
    } else {
        cleanupRunning = false;
        return -1;
    }
}
```

* 根据连接的引用计数来计算空闲连接数和活跃连接数，然后标记出空闲的连接。
* 注释2处，如果空闲连接keepAlive时间超过5分钟，或者空闲连接数超过5个，则从Deque中移除此连接。
* 如果空闲连接大于0，则返回此连接即将到期的时间；
* 如果都是活跃链接，并且大于0，则返回默认的keepAlive时间5分钟
* 如果没有任何连接，则跳出循环并返回-1。

上面通过pruneAndGetAllocationCount方法来判断链接是否活跃，如果返回值大于0则表示活跃连接，否则就是空闲连接。

```java
private int pruneAndGetAllocationCount(RealConnection connection, long now) {
    List<Reference<StreamAllocation>> references = connection.allocations;
    for (int i = 0; i < references.size(); ) {
      Reference<StreamAllocation> reference = references.get(i);
      if (reference.get() != null) {
        i++;
        continue;
      }
      // We've discovered a leaked allocation. This is an application bug.
      StreamAllocation.StreamAllocationReference streamAllocRef =
          (StreamAllocation.StreamAllocationReference) reference;
      String message = "A connection to " + connection.route().address().url()
          + " was leaked. Did you forget to close a response body?";
      Platform.get().logCloseableLeak(message, streamAllocRef.callStackTrace);
      references.remove(i);
      connection.noNewStreams = true;
      // If this was the last allocation, the connection is eligible for immediate eviction.
      if (references.isEmpty()) {
        connection.idleAtNanos = now - keepAliveDurationNs;
        return 0;
      }
    }
    return references.size();
}
```

* 遍历传进来的RealConnection对象的StreamAllocation列表，如果StreamAllocation正在被使用，则接着变量下一个StreamAllocation；如果未被使用，则从列表移除。
* 如果引用为空，则表示此连接没有引用了，这时返回0，表示此连接是空闲连接；否则返回非0，表示此连接时活跃连接。

引用计数是在RealConnection中的allocations集合维护的：

```java
public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();
```

在StreamAllocation中，创建流需要显查到有没有可用的RealConnection，如果是new 的RealConnection对象时，就需要调用acquire方法的，里面会调用：

```java
connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
```

将当前的流添加到allocations结合中去，当释放流时，会调用release方法，remove掉该元素。

RealConnection是socket的物理连接的包装，List中的StreamAllocation数量也是socket被引用的计数。

来看connectionBecameIdle方法，当有连接空闲时，唤起cleanup线程清洗连接池：

```java
  boolean connectionBecameIdle(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (connection.noNewStreams || maxIdleConnections == 0) {
      connections.remove(connection);
      return true;
    } else {
      notifyAll(); // Awake the cleanup thread: we may have exceeded the idle connection limit.
      return false;
    }
  }
```

来看deduplicate方法。

```java
@Nullable Socket deduplicate(Address address, StreamAllocation streamAllocation) {
  assert (Thread.holdsLock(this));
  for (RealConnection connection : connections) {
    if (connection.isEligible(address, null)
        && connection.isMultiplexed()
        && connection != streamAllocation.connection()) {
      return streamAllocation.releaseAndAcquire(connection);
    }
  }
  return null;
}
```

该方法主要是针对HTTP/2场景下多个多路复用连接清除的场景。如果是当前连接是HTTP/2，那么所有指向该站点的请求都应该基于同一个TCP连接。

***

### 参考链接

[OKHttp源码解析(九)](https://www.jianshu.com/p/6166d28983a2)

[okhttp之StreamAllocation](https://www.jianshu.com/p/fbbb018ceb64)

