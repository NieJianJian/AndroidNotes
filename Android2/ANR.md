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

有以下四种场景：

 （1）Service Timeout：Service在特定的时间内无法处理完成；

 （2）BroadcastQueue Timeout：BroadcastReceiver在特定时间内无法处理完成

 （3）ContentProvider Timeout：内容提供者执行超时

 （4）inputDispatching Timeout: 按键或触摸事件在特定时间内无响应。

最终会通过AppErrors调用AppNotRespondingDialog.show()来弹出对话框。

### 参考文献

1. [ANR监测机制](https://www.jianshu.com/p/ad1a84b6ec69)

