## 第6章 理解ActivityManagerService

### 1 AMS家族

### 1.1 Android 7.0的AMS家族

`ActivityManager`是一个和AMS相关联的类。`ActivityManager`中的方法会通过`ActivityManagerNative`（简称AMN）的`getDefault`方法来得到`ActivityManagerProxy`（简称AMP），通过AMP就可以和AMN进行通信，而AMN是一个抽象类，它将功能交由它的子类AMS来处理。因此**AMP就是AMS的代理类**。

* 在Activity的启动过程中调用了`Instrumentation`的`execStartActivity`方法，里面有如下代码：

  ```java
  int result = ActivityManagerNative.getDefault().startActivity(...);
  ```

* 在`execStartActivity`中调用AMN的`getDefault`来获取AMS的代理类AMP，然后调用AMP的`startActivity`方法。

  ```java
  static public IActivityManager getDefault() { return gDefault.get(); }
  private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
      protected IActivityManager create() {
          IBinder b = ServiceManager.getService("activity"); // 1
          IActivityManager am = asInterface(b); // 2
          return am;
      }
  };
  ```

  `gDefault`是一个Singleton类。

  * 注释1处得到名为"activity"的Service的引用，也就是IBinder类型的AMS的引用。
  * 注释2处将得到的引用封装成AMP类型对象，并将它保存到`gDefault`中，此后调用AMN的`getDefault`方法就可以获得AMS的代理对象AMP。

* 上述代码中，注释2处的`asInterface`代码如下：

  ```java
  static public IActivityManager asInterface(IBinder obj) {
      if (obj == null) { return null; }
      IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor); // 1
      if (in != null) { return in; }
      return new ActivityManagerProxy(obj); // 2
  }
  ```

  * `descriptor`的值为`android.app.IActivityManager`
  * 注释1处的代码用来查询本地进程中是否有IActivityManager接口的实现，如果有则返回，如果没有就在注释2处将IBinder类型的AMS引用封装成AMP。

* AMP的构造函数如下：

  ```java
  class ActivityManagerProxy implements IActivityManager {
      public ActivityManagerProxy(IBinder remote) {
          mRemote = remote;
      }
  }
  ```

  * AMP的构造方法中，将AMS的引用赋值给变量`mRemote`，这样AMP中就可以使用AMS了。
  * `IActivityManager`是一个接口，AMN和AMP都实现了这个接口，用于实现代理模式和Binder通信。

* AMP是AMN的内部类。回到AMP的`startActivity`方法中，有如下一行代码：

  ```java
  mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
  ```

  首先将请求参数写入到`Parcel`类型的`data`中。通过`IBinder`类型对象`mRemote`（AMS的引用）向服务端的AMS发送一个`START_ACTIVITY_TRANSACTION`类型的进程间通信请求。那么服务端AMS就会从Binder线程池中读取到客户端发来的数据，最终会调用AMN的`onTransact`方法。

* AMN的`onTransact`方法中，处理消息的过程如下：

  ```java
  case START_ACTIVITY_TRANSACTION:
  {
      ...
      int result = startActivity(app, callingPackage, intent, resolvedType,
          resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
      reply.writeNoException();
      reply.writeInt(result);
      return true;
  }
  ```

* 然后调用到了AMS的`startActivity`方法。

* 然后调用AMS的`startActivityAsUser`方法。

AMP是AMN的内部类，它们都实现了IActivityManager接口，这样他们就可以实现代理模式，具体来讲是**远程代理**：AMP和AMN是运行在两个进程中的，AMP是Client端，AMN是Server端，而Server端中具体的功能都是由AMN的子类AMS来实现的。因此，**AMP就是AMS在Client端的代理类**。AMN又实现了Binder类，这样AMP和AMS就可以通过Binder来进行进程间通信。ActivityManager通过AMN的`getDefault`方法得到AMP，通过AMP就可以和AMS进行通信了。

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/ams_android7family.png" alt="Android 7.0 AMS家族" style="zoom:80%;" />

![AMP和AMS进行通信](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/ams_android7_amsandamp.png)

除了ActivityManager外，想要和AMS通信都需要经过AMP。

### 1.2 Android 8.0的AMS家族

`ActivityManager`的`getService`方法，如下所示：

```java
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

* 注释1处得到名为"activity"的Service引用，也就是IBinder类型的AMS的引用。
* 注释2处将它换成IActivityManager类型的对象，这段代码采用的是AIDL，IActivityManager是由AIDL工具在编译时自动生成的。

要实现进程间通信，服务器端也就是AMS只需要继承IActivityManager.Stub类并实现相应的方法就可以了。

![Android 8.0 AMS家族](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/ams_android8family.jpg)

***

### 2 AMS的启动过程

AMS的启动是在`SystemServer`进程中启动的。

* `SystemServer`的`main`方法中，调用了`run`方法。

* `run`方法中调用`startBootstrapServices`方法启动引导服务。

* `startBootstrapServices`方法中

  ```java
  mActivityManagerService = mSystemServiceManager.startService(
      ActivityManagerService.Lifecycle.class).getService(); // 1
  ```

* 调用`SystemServiceManager`的`startService`方法，参数为`ActivityManagerService.Lifecycle.class`。

  ```java
  public void startService(@NonNull final SystemService service) {
      mServices.add(service); // 1
      try {
          service.onStart(); // 2
      } 
  }
  ```

  * 注释1处将`service`对象添加到ArrayList类型的`mServices`中来完成注册。
  * 注释1处调用`service`的`onStart`方法来启动`service`对象。

* `Lifecycle`是AMS的内部类。

  ```java
  public static final class Lifecycle extends SystemService {
      private final ActivityManagerService mService;
      public Lifecycle(Context context) {
          super(context);
          mService = new ActivityManagerService(context); // 1
      }
      @Override
      public void onStart() {
          mService.start(); // 2
      } 
      public ActivityManagerService getService() {
          return mService; // 3
      } 
  }
  ```

  * 注释1处，在`Lifecycle`的构造方法中创建了AMS实例。
  * 当调用`SystemService`的`onStart`方法时，实际上调用的AMS的`start`方法。
  * 注释3处的`getService`方法返回AMS实例。

***

### 3 AMS与应用程序进程

Zygote的Java层框架中，会创建一个Server端的Socket，这个Socket用来等待AMS请求Zygote来创建新的应用程序进程。以Service的启动过程为例，Service启动过程中会调用`ActiveServices`的`bringUpServiceLocked`方法。如下所示：

```java
private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
                                    boolean whileRestarting, boolean permissionsReviewRequired)
        throws TransactionTooLargeException {
    ... 
    // 获取Service想要在那个进程运行
    final String procName = r.processName; // 1
    ProcessRecord app;
    if (!isolated) {
        app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false); // 2
        if (app != null && app.thread != null) { // 3
            try {
                app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                realStartServiceLocked(r, app, execInFg); // 4
                return null;
            } catch (TransactionTooLargeException e) {...}
        }
    } else {...}
    // 如果用来运行Service的应用程序进程不存在
    if (app == null && !permissionsReviewRequired) { // 5
        if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                hostingType, r.name, false, isolated, false)) == null) { // 6
            ...
        }
    }
    return null;
}
```

* 注释1处得到`ServiceRecord`类型的`processName`的值，并赋值给`procName`。其中`processName`是用来描述Service想要在哪个进程进行，默认是当前进程，也可在`AndroidManifest`文件中设置了`android:process`属性来开启一个新的线程。
* 注释2处根据`procName`和Service的`uid`来查询是否存在一个与Service对应的`ProcessReocrd`类型的对象`app`。`ProcessRecord`主要用来描述运行的应用程序进程的信息。
* 注释5处判断如果Service对应的应用程序进程不存在，就调用注释6处的AMS的方法来创建对应的应用程序进程。
* 注释3处判断如果应用程序进程存在，则调用注释4处的`realStartServiceLocked`方法来启动Service。

AMS与应用程序进程的关系主要有以下两点：

* 启动应用程序进程时AMS会检查这个应用程序需要的应用程序进程是否存在。
* 如果需要的应用程序进程不存在，AMS就会请求Zygote进程创建需要的应用程序进程。

***

### 4 AMS重要的数据结构

### 4.1 解析ActivityRecord

**作用**：用来描述一个Activity，内部记录了Activity的所有信息。

**创建时机**：启动Activity时被创建，在`ActivityStarter`的`startActivity`方法中创建。

**表**：ActivityRecord的部分重要成员变量。如下：

| 名称                | 类型                   | 说明                                                         |
| ------------------- | ---------------------- | ------------------------------------------------------------ |
| service             | ActivityManagerService | AMS的引用                                                    |
| info                | ActivityInfo           | Activity中代码和AndroidManifest设置的节点信息，比如launchMode |
| launchedFromPackage | String                 | 启动Activity的包名                                           |
| taskAffinity        | String                 | Activity希望归属的栈                                         |
| task                | TaskRecord             | ActivityRecord所在TaskRecord                                 |
| app                 | ProcessRecord          | ActivityRecord所在的应用程序进程                             |
| state               | ActivityState          | 当前Activity的状态                                           |
| icon                | init                   | Activity的图标资源标识符                                     |
| theme               | init                   | Activity的主题资源标识符                                     |

### 4.2 解析TaskRecord

**作用**：用来描述一个Activity任务栈。

**表**：TaskRecord的部分重要成员变量。如下：

| 名称        | 类型                       | 说明                           |
| ----------- | -------------------------- | ------------------------------ |
| taskId      | int                        | 任务栈的唯一标识符             |
| affinity    | String                     | 任务栈的倾向性                 |
| intent      | Intent                     | 启动这个任务栈的Intent         |
| mActivities | ArrayList< ActivityRecord> | 按照历史顺序排列的Activity记录 |
| mStack      | ActivityStack              | 当前归属的ActivityStack        |
| mService    | ActivityManagerService     | AMS引用                        |

### 4.3 解析ActivityStack

**作用**：用来管理系统所有Activity，其内部维护了Activity的所有状态、特殊状态的Activity以及和Activity相关的列表等数据。

`ActivityStack`是由`ActivityStackSupervisor`来进行管理。

`ActivityStackSupervisor`是在AMS的构造方法中被创建的。

* **1. ActivityStack的实例类型**

  在`ActivityStackSupervisor`中有多种`ActivityStack`实例，如下：

  ```java
  public class ActivityStackSupervisor {
      ActivityStack mHomeStack;
      ActivityStack mFocusedStack;
      private ActivityStack mLastFocusedStack;
  }
  ```

  * `mHomeStack`用来存储Launcher APP的所有Activity
  * `mFocusedStack`表示当前正在接收输入或启动下一个Activity的所有Activity。
  * `mLastFocusedStack`表示此前接收输入的所有Activity。

  假设要获取`mFocusedStack`，只需要调用：

  ```java
  ActivityStack getFocusedStack() { return mFocusedStack; }
  ```

* **2. ActivityState**

  在`ActivityStack`中通过枚举存储了Activity的所有状态，如下所示：

  ```java
  enum ActivityState {
      INITIALIZING,
      RESUMED,
      PAUSING,
      PAUSED,
      STOPPING,
      STOPPED,
      FINISHING,
      DESTROYING,
      DESTROYED
  }
  ```

  **使用场景**：比如`overridePendingTransition`方法，是用来设置Activity的切换动画。只有`ActivityState`为`RESUMED`状态或者`PAUSING`状态时才会调用WMS类型的`mWindowManager`对象的`overridePendingAppTransition`方法来切换动画。

* **3. 特殊状态的Activity**

  在`ActivityStack`中定义了一些特殊状态的Activity，如下所示：

  ```java
  ActivityRecord mPausingActivity = null; // 正在暂停的Activity
  ActivityRecord mLastPausedActivity = null; // 上一个已经暂停的Activity
  ActivityRecord mLastNoHistoryActivity = null; // 最近一次没有历史记录的Activity
  ActivityRecord mResumedActivity = null; // 已经Resume的Activity
  // 传递给convertToTranslucent方法最上层的Activity    
  ActivityRecord mTranslucentActivityWaiting = null; 
  ```

* **4. 维护的ArrayList**

  在`ActivityStack`中维护了很多ArrayList，如表所示：

  | ArrayList          | 元素类型       | 说明                                                         |
  | ------------------ | -------------- | ------------------------------------------------------------ |
  | mTaskHistory       | TaskRecord     | 所有没有被销毁的Activity任务栈                               |
  | mLRUActivities     | ActivityRecord | 正在运行的Activity，列表中的第一个条目是最近最少使用的Activity |
  | mNoAnimActivities  | ActivityRecord | 不考虑转换动画的Activity                                     |
  | mValidateAppTokens | TaskGroup      | 用于与窗口管理器验证应用令牌                                 |

***

### 5 Activity栈管理

### 5.1 Activity任务栈模型

![Activity任务栈模型](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/activitytaskrecordmodel.png)

* ActivityRecord用来记录一个Activity的所有信息。
* TaskRecord中包含一个或多个ActivityReord。
* TaskRecord用来表示Activity的任务栈，用来管理栈中的ActivityRecord。
* ActivityStack又包含了一个或多个TaskRecord，它是TaskRecord的管理者。

**优点**：有了栈管理，应用可以复用自身应用中以及其他应用中的Activity，节省了资源。

**eg**：一款社交应用中，这个应用的联系人详情页面提供了联系人的邮箱，我们点击邮箱时会跳到发送邮件的界面。社交应用和系统Email中的Activity处于不同应用程序进程的，而有了栈管理，就可以把发送邮件界面放到社交应用中详情页面所在栈的栈顶，来做跨进程操作。

### 5.2 Launch Mode

**作用**：设定Activity的启动模式。无论是那种启动模式，所启动的Activity都会位于Activity栈的栈顶。

* **standerd**：默认模式，每次启动Activity都会创建一个新的Activity实例。
* **singleTop**：
  * 如果要启动的Activity已经在栈顶，则不会重新创建，调用该Activity的`onNewIntent`方法。
  * 如果要启动的Activity不在栈顶，则会重新创建该Activity实例。
* **singleTask**：
  * 如果要启动的Activity已经位于想要归属的栈，那就不会重新创建实例，将栈中位于该Activity之上的所有Activity出栈，并调用该Activity的`onNewIntent`方法。
  * 如果要启动的Activity不存在与它想要归属的栈中，并且栈存在， 则创建该Activity实例。
  * 如果要启动的Activity想要归属的栈不存在，则首先要创建一个新栈，然后创建该Activity实例并压入栈中。
* **singleInstance**：先创建新栈，然后创建Activity实例并压入新栈中，新栈中只会存在这一个Activity实例。

### 5.3 Intent的FLAG

FLAG也可以设定Activity的启动方式。并且优先级满足`FALG > Launch Mode`.

* **FLAG_ACTIVITY_SINGLE_TOP**：和`singleTop`效果一样。
* **FLAG_ACTIVITY_NEW_TASK**：和`singleTask`效果一样。

### 5.4 taskAffinity

**作用**：用来指定Activity希望归属的栈。默认情况下，同一个应用程序所有的Activity都有着相同的`taskAffinity`。

`taskAffinity`在下面两种情况时会产生效果。

* (1) `taskAffinity`与`FLAG_ACTIVITY_NEW_TASK`或者`singleTask`配合。如果新启动Activity的`taskAffinity`和栈`taskAffinity`相同则加入到该栈中；如果不同，就会创建新栈。
* (2) `taskAffinity`与`allowTaskReparenting`配合。如果`allowTaskReparenting`为true，说明Activity具有转移的能力。拿之前的发邮件为例，当社交应用启动了发送邮件的Activity，此时发送邮件的Activity是和社交应用处于同一个栈中的，并且这个栈位于前台。如果发送邮件的Activity的`allowTaskReparenting`设置为true，此后Email应用所在的栈位于前台时，发送邮件的Activity就会由社交应用的栈中转移到与它更亲近的邮件应用（taskAffinity相同）所在的栈中。