* Android 8.0
* [源码地址：frameworks/base/core/java/com/android/internal/os/ZygoteInit.java](https://www.androidos.net.cn/android/8.0.0_r4/xref/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java)

```java
public static void main(String argv[]) {
    ......
    try {
        ......
        //创建一个Server端的socket，socketName = “zygote”
        //在这个socket上会等待AMS的请求
        zygoteServer.registerServerSocket(socketName); // 1
        if (!enableLazyPreload) {
            bootTimingsTraceLog.traceBegin("ZygotePreload");
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                SystemClock.uptimeMillis());
            //预加载类和资源
            preload(bootTimingsTraceLog); // 2
            EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                SystemClock.uptimeMillis());
            bootTimingsTraceLog.traceEnd(); // ZygotePreload
        } else {
            Zygote.resetNicePriority();
        }
        //启动SystemServer进程
        if (startSystemServer) {
            startSystemServer(abiList, socketName, zygoteServer); // 3
        }
        //等待AMS请求，里面是一个while(true)循环
        zygoteServer.runSelectLoop(abiList); // 4
        zygoteServer.closeServerSocket();
    }......
}
```

