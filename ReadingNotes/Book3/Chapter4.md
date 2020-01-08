## 第4章 四大组件的工作过程

### 1 根Activity的启动过程

根Activity的启动过程分为3个部分来讲，分别是Launcher请求AMS过程、AMS到ApplictionThread的调用过程和ActivityThread启动Activity过程。

### 1.1 Launcher请求AMS过程

![Launcher请求AMS的时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/rootactivitystart_1.png)

* 点击应用快捷图标，会通过Launcher请求AMS启动该应用程序。调用Launcher的`startActivitySafely`方法。该方法中，创建新的任务栈，这样根Activity就可以在新的任务栈中启动。

  ```java
  intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
  ```

* 调用`Activity`的`startActivity`方法

  ```java
  @Override
  public void startActivity(Intent intent, @Nullable Bundle options) {
      if (options != null) {
          startActivityForResult(intent, -1, options);
      } else {
          startActivityForResult(intent, -1);
      }
  }
  ```

  第二个参数为-1，表示不需要知道Activity启动的结果。

* ````java
  public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                     @Nullable Bundle options) {
      if (mParent == null) {
          options = transferSpringboardActivityOptions(options);
          Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(
                          this, mMainThread.getApplicationThread(), mToken, this,
                          intent, requestCode, options);
          ...
      }
  }
  ````

  `mParent`表示当前Activity的父类。由于目前根Acitivity还未创建出来，所以为null。

* 接着调用`Instrumentation`的`execStartActivity`方法。`Instrumentation`主要用来监控应用程序与系统的交互。该方法中有如下一行代码，

   ```java
  ActivityManager.getService().startActivity(...);
  ```

  * 首先调用`ActivityManager`的`getService`方法来获取AMS的代理对象，然后调用`startActivity`。
  * Android 8.0之前是通过`ActivityManagerNative`的`getDefault`来获取AMS代理对象的。

* ```java
  @frameworks/base/core/java/android/app/ActivityManager.java
  public static IActivityManager getService() {
      return IActivityManagerSingleton.get();
  }
  
  private static final Singleton<IActivityManager> IActivityManagerSingleton =
          new Singleton<IActivityManager>() {
      @Override
      protected IActivityManager create() {
          final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE); // 1
          final IActivityManager am = IActivityManager.Stub.asInterface(b); // 2
          return am;
      }
  };
  ```

  * `getService`方法调用了`IActivityManagerSingleton`的`get`方法。

  * 注释1处得到名为"activity"的Service引用，也就是IBinder类型的AMS的引用。

  * 注释2处通过AIDL的方式将它转换成`IActivityManager`对象。

    > `IActivityManager.java`类是由AIDL工具在编译时自动生成的，`IActivityManager.aidl`的文件路径为`frameworks/base/core/java/android/app/IActivityManager.aidl`。要实现进程间通信，服务器端也就是AMS只需要继承`IActivitiyManager.Stub`类并实现相应的方法就可以。
    >
    > 注：Android8.0之前用AMS的代理对象`ActivityManagerProxy`来与AMS进程间通信。

* 从上面得知，`Instrumentation`类的`execStartActivity`方法中，最终调用的AMS的`startActivity`方法。

### 1.2 AMS到ApplicationThread的调用过程

![AMS到ApplicationThread的调用过程的时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/rootactivitystart_2.png)

* AMS的`startActivity`方法中，调用了`startActivityAsUser`方法，后者比前者多一个参数`UserHandle.getCallingUserId()`，获取调用者的UserId，用来确定调用者的权限。

  * 判断调用者进程是否被隔离，如果是则抛出`SecurityException`异常。
  * 调用者如果没有权限，抛出`SecurityException`异常。

* 随后调用`ActivityStarter`的`startActivityLocked`方法，该方法的：

  * 倒数第二个参数类型为**`TaskRecord`，代表启动的Activity所在的栈**。
  * 倒数第一个参数`"startActivityAsUser"`代表启动的理由。

  > ActivityStarter是Android 7.0新加入的类，它是加载Activity的控制类，会收集所有的逻辑来决定如何将Intent和Flags转换为Activity，并将Activity和Task以及Stack相关联。
  * 该方法中，判断启动理由不为空，如果空则抛出`IllegalArgumentException`异常。

    ```java
    if (TextUtils.isEmpty(reason)) {
        throw new IllegalArgumentException("Need to specify a reason.");
    }
    ```

* 然后调用`ActivityStarter`的`startActivity`方法，该方法内容较多，主要做了以下操作：

  * 判断`IApplicationThread`类型的`caller`是否为null，这个`caller`指向的是Launcher所在的应用程序进程的`ApplicationThread`对象。
  * 调用AMS的`getRecordForAppLocked`方法得到的是代表Launcher进程的`callerApp`对象，它是`ProcessRecord`类型的，**`ProcessRecord`用来描述一个应用程序进程**。**`ActivityRecord`用来描述一个Activity**，用来记录一个Activity的所有信息。
  * 创建即将要启动的Activity的描述类`ActivityRecord`，赋值给`ActivityRecord[]`类型的`outActivity`。并将`outActivity`传递下去。

* 调用`ActivityStarter`的`startActivityUnchecked`方法，该方法主要**处理与栈管理相关的逻辑**。

  * 此方法执行过程中，有一个判断条件，就是之前对Intent的Flag设置的`FLAG_ACTIVITY_NEW_TASK`，此时条件是满足的，继续执行代码。

  * 创建一个新的**`TaskRecord`，用来描述一个Activity任务栈**。

    > Acitivty任务栈是一个假想的模型，并不真实存在。

* 调用`ActivityStackSupervisor`的`resumeFocusedStackTopActivityLocked`方法

  ```java
  boolean resumeFocusedStackTopActivityLocked(
          ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions){
      // 获取要启动的Activity所在栈的栈顶的不是处于停止状态的ActivityRecord
      final ActivityRecord r = mFocusedStack.topRunningActivityLocked(); // 1
      if (r == null || !r.isState(RESUMED)) { // 2
          mFocusedStack.resumeTopActivityUncheckedLocked(null, null); // 3
      } else if (r.isState(RESUMED)) {
          mFocusedStack.executeAppTransition(targetOptions);
      }
      return false;
  }
  ```

  * 注释1处获取要启动的Activity所在栈的栈顶的不是处于停止状态的ActivityRecord。
  * 注释2处判断ActivityRecord不为null，或者要启动的Activity的状态不是RESUME状态，调用注释3的方法。对于即将要启动的Activity，注释2处的判断条件是肯定满足的。

* 调用`ActivityStack`的`resumeTopActivityUncheckedLocked`方法。

* 调用`ActivityStack`的`resumeTopActivityInnerLocked`方法。

* 调用`ActivityStackSupervisor`的`startSpecificActivityLocked`方法。

  ```java
  void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
      // 获取即将启动的Activity所在的应用程序进程
      ProcessRecord app = mService.getProcessRecordLocked(r.processName,
              r.info.applicationInfo.uid, true); // 1
      if (app != null && app.thread != null) { // 2
          try {
              if ((r.info.flags& ActivityInfo.FLAG_MULTIPROCESS) == 0
                      || !"android".equals(r.info.packageName)) {
                  app.addPackage(r.info.packageName, r.info.applicationInfo.longVersionCode,
                          mService.mProcessStats);
              }
              realStartActivityLocked(r, app, andResume, checkConfig); // 3
              return;
          } catch (RemoteException e) {
              Slog.w(TAG, "Exception when starting activity "
                      + r.intent.getComponent().flattenToShortString(), e);
          }
      }
      mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
              "activity", r.intent.getComponent(), false, false, true);
  }
  ```

  * 注释1处获取即将启动的Activity所在的应用程序进程。
  * 注释2处判断该进程是否已经启动。如果已经运行，执行注释3。

* 调用`ActivityStackSupervisor`的`realStartActivityLocked`方法。

  ```java
  final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
                boolean andResume, boolean checkConfig) throws RemoteException {
      ...
          app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                  System.identityHashCode(r), r.info,
                  mergedConfiguration.getGlobalConfiguration(),
                  mergedConfiguration.getOverrideConfiguration(), r.compat,
                  r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                  r.persistentState, results, newIntents, !andResume,
                  mService.isNextTransitionForward(), profilerInfo);
      ...
  }
  ```

  * `app.thread`指的是`IApplicationThread`，它的实现是`ActivityThread`的内部类`ApplicationThread`，其中`ApplicationThread`继承了`IApplicationThread.Stub`。
  * `app`指的是传入的要启动的Activity所在的应用程序进程，所以这段代码指的是要在目标应用程序进程启动Activity。

  当前代码逻辑运行在AMS所在的进程（SystemServer进程）中，通过ApplicationThread来与应用程序进程Binder通信，换句话说，ApplicationThread是AMS所在进程（SystemServer进程）和应用程序进程的通信桥梁。如下图所示。

  ![AMS与应用程序进程通信](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/rootactivitystart_amsandprocess.png)]

### 1.3 ActivityThread启动Activity的过程

![ActivityThread启动Activity过程的时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/rootactivitystart_3.png)

> ApplicationThread是ActivityThread的内部类。应用程序进程创建后会运行代表主线程的实例ActivityThread，它管理着当前应用程序进程的主线程。

* `ApplicationThread`的`scheduleLauncherActivity`方法中，将启动Activity的参数封装成`ActivityClientRecord`。

* `AcitivityThread`的`sendMessage`方法向`H`类发送类型为`LAUNCH_ACTIVITY`的消息，并将`ActivityClientRecord`作为参数传递过去。

  > `H`类是`ActivityThread`的内部类并继承自`Handler`，是应用程序进程中主线程的消息管理类。

  因为`ApplicationThread`是一个`Binder`，它的调用逻辑运行在Binder线程池中，所以这里需要用`H`将代码的逻辑切换到主线程中。

* 调用`H`的`handleMessage`方法，代码如下：

  ```java
  public void handleMessage(Message msg) {
      switch (msg.what) {
          case LAUNCH_ACTIVITY:
              Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
              final ActivityClientRecord r = (ActivityClientRecord) msg.obj; // 1
              r.packageInfo = getPackageInfoNoCheck(
                      r.activityInfo.applicationInfo, r.compatInfo); // 2
              handleLaunchActivity(r, null, "LAUNCH_ACTIVITY"); // 3
              Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
              break;
      }
  }
  ```

  * 注释1处将传过来的msg的成员变量obj转换为`ActivityClientRecord`。

  * 注释2处获得`LoadedApk`类型的对象并赋值给`r.packageInfo`变量。

    > 应用程序要启动Activity时需要将该Activity所属的APK加载进来，而`LoadedApk`就是用来描述已加载的APk文件的。

* 调用`handleLaunchActivity`方法，方法内一些简略代码如下：

  ```java
  Activity a = performLaunchActivity(r, customIntent); // 1
  if (a != null) {
      // 将Activity的状态置为Resume
      handleResumeActivity(r.token, ...); // 2    
  } else {
      // 停止Activity启动
      ActivityManager.getService().finishActivity(r.token, ...);
  }
  ```

  * 调用`performLaunchActivity`方法启动Activity
  * 如果`a`不为null，将Activity的状态置为Resume。
  * 如果`a`为null，通知AMS停止启动Activity。

* 调用`ActivityThread`的`performLaunchActivity`方法来启动Activity。

  ```java
  private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
      ActivityInfo aInfo = r.activityInfo; // 1 获取ActivityInfo类
      if (r.packageInfo == null) {
          r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                  Context.CONTEXT_INCLUDE_CODE); // 2 获取APK文件的描述类LoadedApk
      }
      ComponentName component = r.intent.getComponent(); // 3
      ...
      // 创建要启动的Activity的上下文环境
      ContextImpl appContext = createBaseContextForActivity(r); // 4
      Activity activity = null;
      try {
          java.lang.ClassLoader cl = appContext.getClassLoader();
          // 用类加载器来创建该Activity的实例
          activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent); // 5
          ...
      } catch (Exception e) {
          ...
      }
      try {
          // 创建Application
          Application app = r.packageInfo.makeApplication(false, mInstrumentation); // 6
          ...
          if (activity != null) {
              ...
              // 7 初始化Activity
              activity.attach(appContext, this, getInstrumentation(), r.token,
                      r.ident, app, r.intent, r.activityInfo, title, r.parent,
                      r.embeddedID, r.lastNonConfigurationInstances, config,
                      r.referrer, r.voiceInteractor, window, r.configCallback);
              ...
              if (r.isPersistable()) {
                  mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState); // 8
              } else {
                  mInstrumentation.callActivityOnCreate(activity, r.state);
              }
              ...
          }
          r.paused = true;
          mActivities.put(r.token, r);
      } catch (Exception e) {
          ...
      }
      return activity;
  }
  ```
  
  * 注释1处用于获取`ActivityInfo`，用于存储代码以及`AndroidManifests`设置的Activity和Receiver节点信息，比如Activity的`theme`和`launchMode`。
  * 注释2处获取APK文件的描述类`LoadedApk`。
  * 注释3处获取要启动的Activity的`ComponentName`类，在`ComponentName`类中保存了该Activity的包名和类名。
  * 注释4处创建要启动的Activity的上下文环境。
  * 注释5处根据`ComponentName`存储的Activity类名，用类加载器来创建该Activity的实例。
  * 注释6处用来创建Application，`makeApplication`方法内部会调用`Application`的`onCreate`方法。
  * 注释7处调用Activity的`attach`方法初始化Activity，在`attach`方法中会创建Window对象（PhoneWindow）并与Activity自身进行关联。
  
* 注释8处调用`Instrumentation`的`callActivityOnCreate`方法来启动Activity。

* 调用`Activity`的`performCreate`方法。在该方法中，调用了`onCreate`方法，至此，根Activity就启动了，即应用程序就启动了。

### 1.4 根Activity启动过程中涉及的进程

Activity的启动过程中会涉及到4个进程，分别是Zygote进程、Launcher进程、AMS所在进程（SystemServer）、应用程序进程。关系如下图所示：

![根Activity启动过程中涉及的进程之间的关系](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/rootactivitystart_processrelation.png)

* 首先Launcher进程向AMS请求创建根Activity
* AMS判断根Activity所需的应用程序进程是否存在并启动
* 如果不存在就会请求Zygote进程创建应用程序进程。
* 应用程序进程启动后，AMS会启动根Activity。

上图中步骤2（AMS请求Zygote进程创建应用程序进程）采用的是Socket通信。

步骤1（Launcher进程请求AMS创建根Activity）和步骤4（AMS请求ApplicationThread创建根Activity）采用的是Binder通信。

下图是4个进程调用的时序图：

![根Activity启动过程中进程调用时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/rootactivitystart_processcall.png)

***