## Activity生命周期

下面来针对一些常见的Activity的生命周期情况

### 1. A Activity打开B Activity的生命周期回调

* B Activity的`launchMode`为`standard`或者B没有可复用的实例时：

  ```
  A.onPause -> B.onCreate - > B.onStart -> B.onResume -> A.Stop
  ```

* B Activity的`launchMode`为`singleTop`并且B已经在栈顶时：

  ```
  B.onPause -> B.onNewIntent -> B.onResume
  ```

* B Activity的`launchMode`为`singleTask`并且B有可复用的实例时：

  ```
  A.onPause -> B.onNewIntent -> B.onRestart -> B.onStart -> B.onResume -> A.onStop -> A.onDestory
  ```

  A在栈中的位置如果位于B的上层，A就会被销毁，从而调用A Activity 的 *onDestory* 方法。

### 2. 弹出Dialog对生命周期的影响

我们知道，生命周期回调都是 AMS 通过 Binder 通知应用进程调用的；而弹出 Dialog、Toast、PopupWindow 本质上都直接是通过 WindowManager.addView() 显示的（没有经过 AMS），所以不会对生命周期有任何影响。

如果启动一个Theme为Dialog的Activity，生命周期如下：

```
A.onPause -> B.onCreate - > B.onStart -> B.onResume
```

不会调用A Activity的 *onStop*。

* *onPause*：页面失去焦点，不能交互，但是仍然可见时调用；
* *onStop*：页面不可见时才会调用。

### 3. Activity为什么在onResume之后才显示

虽然我们设置 Activity 的布局一般都是在 onCreate 方法里调用 setContentView 。里面是直接调用 window 的 setContentView，创建一个 DecorView 用来包住我们创建的布局。详情如下：

```java
# PhoneWindow.java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } 
    ...
    // 加载布局，添加到 mContentParent
    // mContentParent 又是 DecorView 的一个子布局  
    mLayoutInflater.inflate(layoutResID, mContentParent);
}
```

然而这一步只是加载好了布局，生成一个 ViewTree , 具体怎么把 ViewTree 显示出来，答案就在下面：

```java
# ActivityThread.java
public void handleResumeActivity(...){
    // onResume 回调
    ActivityClientRecord r = performResumeActivity(...)
    final Activity a = r.activity;
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        ViewManager wm = a.getWindowManager();
        wm.addView(decor, l);// 重点
    }
}
```

WindowManager 的 addView 方法最终将 DecorView 添加到 WMS ，实现绘制到屏幕、接收触屏事件。具体的调用链如下：

```
   WindowManagerImpl.addView
-> WindowManagerGlobal.addView
-> ViewRootImpl.setView     
-> ViewRootImpl.requestLayout() // 执行 View 的绘制流程
   // 通过 Binder 调用 WMS ，WMS 会添加一个 Window 相关的对象
   // 应用端通过 mWindowSession 调用 WMS
   // WMS 通过 mWindow (一个 Binder 对象) 调用应用端  
   mWindowSession.addToDisplay(mWindow) 
```

综上，在 onResume 回调之后，会创建一个 ViewRootImpl ，有了它之后应用端就可以和 WMS 进行双向调用了。 

### 4. onActivityResult的调用时机

A Activity通过`startActivityForResult`启动B Activity，B中返回A的时候生命周期如下：

```
B.onPause -> A.onActivityResult -> A.onRestart -> A.onStart -> A.onResume -> B.onStop
```

可以看出，onActivityResult回调优先于该Activity的所有生命周期。

其实onActivityResult并不属于Activity的生命周期，在该方法的注释中也有如下内容：

```
You will receive this call immediately before onResume() when your activity is re-starting.
```

### 5. 生命周期方法中写死循环会不会导致ANR

先来看下 ANR 的发生场景：

1. Service TimeOut:  service 未在规定时间执行完成；前台服务 20s，后台 200s
2. BroadCastQueue TimeOut: 未在规定时间内未处理完广播；前台广播 10s 内, 后台 60s 内
3. ContentProvider TimeOut:  publish 在 10s 内没有完成
4. Input Dispatching timeout:  5s 内未响应键盘输入、触摸屏幕等事件

在生命周期方法中执行死循环，并不会直接导致 ANR ，只不过死循环阻塞了主线程，如果系统在发生上述的四种情况，就会导致无法在相应的时间内处理事件，从而发生 ANR。

### 6. onSaveInstanceState / onRestoreInstanceState调用时机

在开发者模式中，勾选上 [不保留活动] 选项，就可以随时模拟Activity被意外销毁的场景。

1. 当A Activity 启动 B Activity，A进入后台，如果系统不足时，A可能被销毁，也就是非自愿销毁（自愿销毁：按下返回键或者调用finish()方法）的情况下，会调用onSaveInstanceState方法，并且在从B返回A的时候，会调用onRestoreInstanceState方法。
2. 用户旋转屏幕，Activity被破坏并重建，会调用onSaveInstanceState / onRestoreInstanceState方法。

* A 启动 B，A被回收，onSaveInstanceState在生命周期中的执行顺序如下：

  ```
  A.onPause -> B.onCreate -> B.onStart -> B.onResume -> A.onStop -> A.onSaveInstaneState
  ```

  我们可以看到，onSaveInstanceState方法在onStop之后调用，为什么是这样呢？我们来看下源码：

  ```java
  # ActivityThread.java
  public void handleStopActivity(IBinder token, boolean show, int configChanges, ... ) {
      ...
      performStopActivityInner(r, stopInfo, show, true /* saveState */, finalStateRequest,
              reason);
  ```

  ```java
  # ActivityThread.java
  public void performStopActivityInner(IBinder token, boolean show, int configChanges, ... ) {
      ...
      callActivityOnStop(r, saveState, reason);
  ```

  ```java
  # ActivityThread.java
  private void callActivityOnStop(ActivityClientRecord r, boolean saveState, String reason) {
      ...
      r.activity.performStop(false /*preserveWindow*/, reason); // 1
      ...
      if (shouldSaveState && !isPreP) {
          callActivityOnSaveInstanceState(r); // 2
      }
  }
  ```

  在 ActivityThread 的 callActivityOnStop 方法中，注释1处最终调用了Activity 的 onStop方法，注释2处最终会调用onSaveInstanceState方法，所以 onSaveInstanceState 在 onStop 方法之后调用。

* 当从B返回A时，onRestoreInstanceState在生命周期中的执行顺序如下：

  ```
  B.onPause -> A.onCreate -> A.onStart -> A.onRestoreInstanceState -> A.onResume -> B.onStop
  ```

  我们可以看到，onRestoreInstanceState 方法在 onStart 之后调用，为什么是这样呢？我们来看下源码：

  ```java
  # ActivityThread.java
  public void handleStartActivity(ActivityClientRecord r, PendingTransactionActions pendingActions) {
      ...
      activity.performStart("handleStartActivity"); // 1
      ...
      if (pendingActions.shouldRestoreInstanceState()) {
          if (r.isPersistable()) {
              if (r.state != null || r.persistentState != null) {
                  mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                          r.persistentState); // 2
              }
          } else if (r.state != null) {
              mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state); //3 
          }
      }
  ```

  在 ActivityThread 的handleStartActivity 方法中，注释1处最终会调用 Activity 的onStart 方法，注释2或者注释3处最终会调用 onRestoreInstanceState方法，所以 onRestoreInstanceState 方法在 onStart 方法之后调用。

***

### 参考文献

1. [5 道刁钻的 Activity 生命周期面试题](https://mp.weixin.qq.com/s/oFVGHq7h5byrFUfhajM1ww)
2. [onSaveInstanceState()和onRestoreInstanceState()使用详解](https://www.jianshu.com/p/27181e2e32d2)

