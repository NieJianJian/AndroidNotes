## OkHttp源码分析

[基于OkHttp Version 3.12.0](https://square.github.io/okhttp/changelog_3x/#version-3120)

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

