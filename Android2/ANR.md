## ANR

### 1. timeout

```java
# ActivityManagerService.java
  
// How long we wait for a launched process to attach to the activity manager
// before we decide it's never going to come up for real.
static final int PROC_START_TIMEOUT = 10*1000;

// How long we wait for an attached process to publish its content providers
// before we decide it must be hung.
static final int CONTENT_PROVIDER_PUBLISH_TIMEOUT = 10*1000;

// How long we allow a receiver to run before giving up on it.
static final int BROADCAST_FG_TIMEOUT = 10*1000;
static final int BROADCAST_BG_TIMEOUT = 60*1000;

// How long we wait until we timeout on key dispatching.
static final int KEY_DISPATCHING_TIMEOUT = 5*1000;
```

```java
# ActiveServices.java
  
// How long we wait for a service to finish executing.
static final int SERVICE_TIMEOUT = 20*1000;
// How long we wait for a service to finish executing.
static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;
// How long the startForegroundService() grace period is to get around to
// calling startForeground() before we ANR + stop it.
static final int SERVICE_START_FOREGROUND_TIMEOUT = 10*1000;
```

### 2. 场景

ANR 的四种场景：

1. Service TimeOut:  service 未在规定时间执行完成；前台服务 20s，后台 200s
2. BroadCastQueue TimeOut: 未在规定时间内未处理完广播；前台广播 10s 内, 后台 60s 内
3. ContentProvider TimeOut:  publish 在 10s 内没有完成
4. Input Dispatching timeout:  5s 内未响应键盘输入、触摸屏幕等事件

我们可以看到，Activity 的生命周期回调的阻塞并不在触发 ANR 的场景里面，所以并不会直接触发 ANR。只不过死循环阻塞了主线程，如果系统再有上述的四种事件发生，就无法在相应的时间内处理从而触发 ANR。

最终会通过AppErrors调用AppNotRespondingDialog.show()来弹出对话框。

### 参考文献

1. [ANR监测机制](https://www.jianshu.com/p/ad1a84b6ec69)

