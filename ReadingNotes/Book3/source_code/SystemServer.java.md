* Android 8.0
* [源码地址：frameworks/base/services/java/com/android/server/SystemServer.java](https://www.androidos.net.cn/android/8.0.0_r4/xref/frameworks/base/services/java/com/android/server/SystemServer.java)

```java
/**
 * The main entry point from zygote.
 */
public static void main(String[] args) {
    new SystemServer().run();
}
```

main方法只调用了SystemServer的run方法，如下所示：

```java
private void run() {
    try {
        ...
        // 创建消息Looper
        Looper.prepareMainLooper();
        // 加载了动态库libandroid_servers.so
        System.loadLibrary("android_servers"); // 1
        performPendingShutdown();
        // 创建系统Context
        createSystemContext();
        // 创建SystemServiceManager，用于系统服务的创建、启动和生命周期管理
        mSystemServiceManager = new SystemServiceManager(mSystemContext); // 2
        mSystemServiceManager.setRuntimeRestarted(mRuntimeRestart);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        SystemServerInitThreadPool.get();
    } finally {
        traceEnd();  // InitBeforeStartServices
    }
    // Start services.
    try {
        traceBeginAndSlog("StartServices");
        // 启动引导服务
        startBootstrapServices();
        // 启动核心服务
        startCoreServices();
        // 启动其他服务
        startOtherServices();
        SystemServerInitThreadPool.shutdown();
    } catch (Throwable ex) {
        Slog.e("System", "******************************************");
        Slog.e("System", "************ Failure starting system services", ex);
        throw ex;
    } finally {
        traceEnd();
    }
    ...
    // Loop forever.
    Looper.loop();
}
```

