## 第5章 理解上下文Context

### 1 Context的关联类

Context意为上下文，是一个应用程序环境信息的接口。常用的使用场景两大类，分别是：

* 使用Context调用方法，比如启动Activity、访问资源、调用系统级服务等；
* 调用方法时传入Context，比如弹出Toast、创建Dialog等。

> Context数量 = Activity数量 + Service数量 + 1个Application。

![Context的关联类](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/context_guanlianclass.png))

* `ContextImpl`和`ContextWrapper`继承自`Context`。
* `ContextWrapper`中包含Context类型的`mBase`对象，`mBase`指向的是ContextImpl。
* `Context`的相关设计使用了装饰模式，`ContextWrapper`是装饰类，对`ContextImpl`进行包装，`ContextWrapper`起了方法传递的作用，其中的几乎所有方法都是调用`ContextImpl`的相应方法来实现。
* `ContextThemeWrapper`、`Service`、`Application`都是继承自`ContextWrapper`，这样它们都可以通过`mBase`来使用Context的方法，同时它们也是装饰类。
* `ContextThemeWrapper`中包含了主题相关的方法（如`getTheme`方法），因此需要主题的`Activity`继承了`ContextThemeWrapper`，而不需要主题的`Service`继承自`ContextWrapper`。

Context的关联类才用了装饰模式，主要有以下的**优点**：

* 使用者能够更方便地使用Context。
* 如果ContextImpl发生了变化，它的装饰类ContextWrapper不需要做任何修改。
* ContextImpl的实现不会暴露给使用者，使用者也不必关心ContextImpl的实现。
* 通过组合而非集成的方式，拓展ContextImpl的功能，在运行时选择不同的装饰类，实现不同的功能。

***

### 2 Application Context的创建过程

![Application Context的创建过程的时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/context_applicationcreate.png)]

* `ActivityThread`类作为应用程序进程的主线程管理类，它会调用它的内部类`ApplicationThread`的`scheduleLaunchActivity`方法来启动Activity。

* 在`scheduleLaunchActivity`方法中向`H`类发送`LAUNCH_ACTIVITY`消息，目的是将启动Activity的逻辑方法主线程的消息队列中，因为`ApplicationThread`是在Binder线程池中。

* `H`类是`ActivityThread`的内部类，`handleMessage`处理消息时，

  ```java
  case LAUNCH_ACTIVITY:
      final ActivityClientRecord r = (ActivityClientRecord) msg.obj; 
      r.packageInfo = getPackageInfoNoCheck(r.activityInfo.applicationInfo, r.compatInfo);  
      handleLaunchActivity(r, null, "LAUNCH_ACTIVITY"); 
      Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
      break;
  ```

  通过`getPackageInfoNoCheck`方法获取`LoadedApk`类型的对象，并将该对象赋值给`ActivityClientRecord`的成员变量`packageInfo`，**`LoadedApk`是用来描述已加载的APK文件**。

* 调用`ActivityThread`的`handleLaunchActivity`方法。

* 调用`ActivityThread`的`performLaunchActivity`方法，该方法中有如下代码：

  ```java
  Application app = r.packageInfo.makeApplication(false, mInstrumentation);
  ```

  `ActivityClientRecord`的成员变量`packageInfo`是`LoadedApk`类型的。

* 调用`LoadedApk`的`makeApplication`方法，简化代码如下：

  ```java
  public Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation) {
      if (mApplication != null) { return mApplication; } // 1
      Application app = null;
      ...
      try {
          java.lang.ClassLoader cl = getClassLoader();
          ...
          ContextImpl appContext = ContextImpl.createAppContext(
                  mActivityThread, this); // 2
          app = mActivityThread.mInstrumentation.newApplication(
                  cl, appClass, appContext); // 3
          appContext.setOuterContext(app); // 4
      } catch (Exception e) {
          ...
      }
      mActivityThread.mAllApplications.add(app);
      mApplication = app; // 5
      ...
      return app;
  }
  ```

  * 第一次启动时，`mApplication`为null，执行下面相关代码。
  * 创建`ContextImpl`对象实例`appContext`。
  * 注释3处创建Application，
  * 注释4处将Application赋值给ContextImpl中Context类型的成员变量`mOuterContext`，这样ContextImpl中也包含了Application引用。
  * 注释5将Application赋值给LoadedApk的成员变量`mApplication`，这个`mApplication`是Application类型的对象，它用来代表Application Context。

* 调用`Instrumentaion`中的`newApplication(... ,Context context)`方法。

* 调用`Application`的`attach(Context context)`方法。

* 调用`ContextWrapper`的`attachBaseContext`方法

  ```java
  protected void attachBaseContext(Context base) {
      ...
      mBase = base;
  }
  ```

  这个`base`就是一路传递过来的`ContextImpl`，将`ContextImpl`赋值给`ContextWrapper`的Context类型的成员变量`mBase`，这样在`ContextWrapper`中就可以使用Context的方法。

  **Application的`attach`方法的作用就是使Application可以使用Context的方法**。

***

### 3 Application Context的获取过程

* 调用`ContextWrapper`的`getApplicationContext`方法

  ```java
  return mBase.getApplicationContext(); // mBase指的使ContextImpl
  ```

* 调用`ContextImpl`的`getApplicationContext`方法

* 调用`LoadedApk`的`getApplication`方法

  ```java
  return mApplication;
  ```

  在之前的`LoadedApk`的`makeApplication`中，`mApplication`已经被赋值，这样我们就可以获取到Application Context了。

***

### 4 Activity的Context的创建过程

![Activity的Context创建过程的时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/context_activitycreate.png)

* `ActivityThread`的内部类`ApplicationThread`调用`scheduleLaunchActivity`方法启动Activity，

  * 将启动Activity的参数封装成`ActivityClientRecord`。
  * 调用`sendMessage`方法向`H`类发送类型为`LAUNCH_ACTIVITY`的消息，并将`ActivityClientRecord`传递过去。

* 之后会调用到`ActivityThread`的`performLaunchActivity`方法中，简略代码如下：

  ```java
  private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
      ContextImpl appContext = createBaseContextForActivity(r); // 1
      Activity activity = null;
      try {
          java.lang.ClassLoader cl = appContext.getClassLoader();
          activity = mInstrumentation.newActivity(
                  cl, component.getClassName(), r.intent); // 2
      } catch (Exception e) { ... }
      try {
          if (activity != null) {
              appContext.setOuterContext(activity); // 3
              activity.attach(appContext, this, getInstrumentation(),...);  // 4
          }
      } catch (SuperNotCalledException e) { ... }
      return activity;
  }
  ```

  * 注释2处创建Activity实例
  * 注释1处创建Activity的ContextImpl，并且传入注释4处的`attach`方法中。
  * 注释3处将创建的Activity实例赋值给`ContextImpl`中Context类型的成员变量`mOuterContext`，这样ContextImpl也可以访问Activity的变量和方法。

* 查看`Activity`的`attach`方法，简略代码如下：

  ```java
  final void attach(Context context, ActivityThread aThread,
                    Instrumentation instr, IBinder token, int ident,
                    Application application, Intent intent, ActivityInfo info,
                    CharSequence title, Activity parent, String id,
                    NonConfigurationInstances lastNonConfigurationInstances,
                    Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                    Window window, ActivityConfigCallback activityConfigCallback) {
      attachBaseContext(context); // 1
      mFragments.attachHost(null /*parent*/);
      mWindow = new PhoneWindow(this, window, activityConfigCallback); // 2
      mWindow.setWindowControllerCallback(this);
      mWindow.setCallback(this); // 3
      mWindow.setOnWindowDismissedCallback(this);
      mWindow.getLayoutInflater().setPrivateFactory(this);
      ...
      mWindow.setWindowManager(
              (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
              mToken, mComponent.flattenToString(),
              (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0); // 4
      mWindowManager = mWindow.getWindowManager(); // 5
  }
  ```

  * 注释2处创建`PhoneWindow`，它代表应用程序窗口。

    > PhoneWindow在运行中会间接触发很多事件，比如点击、菜单弹出、屏幕焦点变化等事件，这些事件需要转发给与PhoneWindow关联的Activity，转发操作通过`Window.Callback`接口实现，Activity实现了这个接口。

  * 注释3处将当前Activity通过`Window`的`setCallback`方法传递给`PhoneWindow`。

  * 注释4处为`PhoneWindow`设置`WindowManager`

  * 注释5处获取`WindowManager`并赋值给Activity的成员变量`mWindowManager`，这样Activity就可以通过`getWindowManager`方法获取到`WindowManager`。

* 上述代码的注释1处调用了`ContextThemeWrapper`的`attachBaseContext`方法。

* 随后调用`ContextWrapper`的`attachBaseContext`方法，

  ```java
  protected void attachBaseContext(Context base) {
      ...
      mBase = base;
  }
  ```

  `base`是一路传过来的Activity的ContextImpl，将它赋值给`ContextWrapper`的成员变量`mBase`。这样`ContextWrapper`的功能就可以交由`ContextImpl`来处理，比如：

  ```java
  @frameworks/base/core/java/android/content/ContextWrapper.java
  public Resources.Theme getTheme() {
      return mBase.getTheme();
  }
  ```

  我们调用`ContextWrapper`的`getTheme`方法时，其实是调用的`ContextImpl`的方法。

**总结**：在启动Activity的过程中创建`ContextImpl`，并赋值给`ContextWrapper`的成员变量`mBase`。Activity继承自`ContextWrapper`的子类`ContextThemeWrapper`，这样Activity中就可以使用Context中定义的方法了。

***

### 5 Service的Context创建过程

Service的Context创建过程与Activity的Context创建过程类似，是在Service的启动过程中被创建的。

* `ActivityThread`的`scheduleCreateService`方法中，调用`H`类的`sendMessage`，发送`CREATE_SERVICE`消息，并将封装好的Service的启动参数对象`CreateServiceData`传递过去。

* `H`类对消息进行处理，其中调用了`ActivityThread`的`handleCreateService`方法。简略代码如下：

  ```java
  ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
  service.attach(context, this, ...);
  ```

  创建Service的ContextImpl实例，并传递到`Service`的`attach`中。

* `Service`的`attach`方法中调用了`ContextWrapper`的`attachBaseContext`方法

  ```java
  mBase = base;
  ```

  