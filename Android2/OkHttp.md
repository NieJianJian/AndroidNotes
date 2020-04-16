## OkHttp源码分析

[基于OkHttp Version 3.12.0](https://square.github.io/okhttp/changelog_3x/#version-3120)

* 源码：

  [OkHttpClient](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/OkHttpClient.md)、[Dispatcher](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/Dispatcher.md)、[RealCall](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/RealCall.md)、[NamedRunnable](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/NamedRunnable.md)、[Interceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/Interceptor.md)、[RealInterceptorChain](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/RealInterceptorChain.md)、[RetryAndFollowUpInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/RetryAndFollowUpInterceptor.md)、[BridgeInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/BridgeInterceptor.md)、[CacheInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/CacheInterceptor.md)、[ConnectInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/ConnectInterceptor.md)、[CallServerInterceptor](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/source/okhttp/CallServerInterceptor.md)

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

