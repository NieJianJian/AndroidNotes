## 第2章 Android系统启动

### Android系统启动流程

流程图如下：

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/androidsystemloader.png" alt="Android系统启动流程图 " style="zoom:60%;" />

* **1.启动电源以及系统启动**

  当按下电源时引导芯片代码从预定义的地方（固化在ROM）开始执行。加载引导程序BootLoader到RAM，然后执行。

* **2.引导程序BootLoader**

  引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行。

* **3.Linux内核启动**

  当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置时，它首先在系统文件中寻找`init.rc`文件，并**启动init进程**。

* **4.init进程启动**

  * 初始化和启动属性服务，
  * 并且启动Zygote进程。

* **5.Zygote进程启动**

  * 创建Java虚拟机并为Java虚拟机注册JNI方法，
  * 创建服务器端Socket，
  * 启动SystemServer进程。

* **6.SystemServer进程启动**

  * 启动Binder线程池
  * 启动SystemServiceManager，
  * 启动各种系统服务
    * 引导服务	
    * 核心服务
    * 其他服务

* **7.Launcher进程启动**

  被SystemServer进程启动的AMS会启动Launcher，Launcher启动后会将已安装应用的快捷图标显示到界面上。

***

### 1 init进程启动过程

　　init进程，进程号为1。创建了Zygote和属性服务等。源码文件位于system/core/init中。

### 1.1 init进程的入口函数

从init的入口函数[main](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/source_code/init.cpp.md)方法中看出，init进程主要做以下三件事：

* **创建和挂载启动所需的文件目录**

  > 挂载了tmpfs、devpts、proc、sysfs和selinuxfs共5中文件系统，这些都是系统运行时目录，只在系统运行时存在， 系统停止时会消失。

* **初始化和启动属性服务**

  > 类似于Windows上的注册表管理器，采用key-value的形式来记录用户、软件的一些使用信息。即使系统或软件重启，其还是能够根据之前注册表中的记录，进行相应的初始化工作。

* **解析`init.rc`文件，启动Zygote进程**

init的main方法中，还**设置子进程信号处理函数**

> 在注释2处的`signal_handler_init`函数中进行设置。主要用于防止init进程的子进程成为僵尸进程，接收子进程暂停和终止时发出的SIGCHLD信号。

**Q**：假设init进程的子进程Zygote终止了？

**A**：`signal_handler_init`函数内部会调用`handle_signal`函数，最终会找到Zygote进程并移除所有的Zygote进程的信息，再重启Zygote服务的启动脚本（比如init.zygote64.rc）中带有onrestart选项的服务。

### 1.2 解析init.rc

　　源码位置：`system/core/rootdir/init.rc`。它是一个非常重要的配置文件，由Android初始化语言编写的脚本，这种语言主要包含5中类型语句：`Action`、`Command`、`Service`、`Option`和`Import`  。在Android 8.0中对init.rc文件进行了拆分，每个服务对应一个rc文件。Zygote的启动脚本在`init.zygoteXX.rc`中定义。这里以64位处理器为例，`system/core/rootdir/init.zygote64.rc`的代码如下：

```c++
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

　　上述代码的大致意思就是用Service类型语句**通知init进程创建名为zygote的进程**。

```
// Service类型语句格式如下：
service <name> <pathname> [ <argument> ]* //<service的名字><执行程序路径><传递参数>
< option > //option是service的修饰词，影响什么时候启动，如何启动service。
< option >
......
```

　　`init.rc`文件中的Service类型语句采用ServiceParser来进行解析（`system/core/init/service.cpp`）。大致**解析过程**为：根据参数创建出Service对象，然后根据选项域的内容填充Service对象，最后将Service对象加入vector类型的Service链表中。

### 1.3 init启动zygote

在`init.rc`中有如下代码：

```c++
on nonencrypted
    class_start main
    class_start late_start
```

`class_start`是一个Command类型语句，含义是启动classname为`main`的Service。从上述启动脚本的代码中可以得知，**zygote的classname为`main`**。

* **第一步**：`class_start`对应的函数为`builtins.cpp`中的`do_class_start`。`ForEachServiceInClass`函数遍历Service链表，找到classname为`main`的Zygote。

* **第二步**：执行`service.cpp`的`StartIfNotDisabled`函数，判断Service有没有在其对应的rc文件中设置disabled选项，

* **第三步**：如果没有设置disabled选项，执行`service.cpp`的`Start`函数。

  * 首先，判断Service是否已经运行，如果运行，则返回false。

  * 判断需要启动的Service的对应的执行文件是否存在，不存在则不启动。

  * 如果子进程没启动，调用`fork`函数创建子进程，并返回`pid`值。

  * `pid`为0时，表示当前代码逻辑在子进程中执行，然后调用`execve`函数，启动Service子进程，并进入该Service的`main`函数中。

  * 如果该Service是Zygote，则执行程序的路径为`system/bin/app_process64`，对应文件`app_main.cpp`中的`main`函数，执行：

    ```c++
    runtime.start("com.android.internal.os.ZygoteInit", args, zygote)；
    ```

    执行runtime的start函数，至此Zygote就启动了。

### 1.4 属性服务

　　init进程启动时会启动属性服务，并为其分配内存，用来存储这些属性，如果需要这些属性直接读取就可以了。`init.cpp`的main函数中：

```c++
property_init(); // 初始化属性服务
start_property_service(); // 启动属性服务
```

> **epol介绍**：
>
> 在Linux新内核中，`epoll`用来替换`select`，`epoll`是Linux内核为处理大批量文件描述符而做了改进的`poll`，是Linux下多路复用I/O接口select/poll的增强版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率。`epoll`内部用于保存事件的数据类型是红黑树，查找速度快，select采用的数组保存信息，查找速度很慢，只有当等待少量文件描述符时，`epoll`和`select`的效率才差不多。

***

### 2 Zygote进程启动过程

　　Android系统中，DVM和ART、应用程序进程以及运行系统的关键服务的SystemServer进程都是有Zygote进程来创建的，我们也将它称为孵化器。它通过fock（复制进程）的形式来创建应用程序进程和SystemServer进程，由于Zygote进程在启动时会创建DVM或者ART，因此通过fock而创建的应用程序进程和SystemServer进程可以在内部获取一个DVM或者ART的实例副本。

### 2.1 Zygote启动脚本

在`init.rc`中采用Import类型语句来引入Zygote脚本。

```
import /init.${ro.zygote}.rc
```

从Android 5.0开始，Android支持64位程序，Zygote也就有了32位和64位的区别。`ro.zigote`取值如下：

* init.zygote32.rc
* init.zygote32_64.rc
* init.zygote64.rc
* init.zygote64_32.rc

这些启动脚本放在`system/core/rootdir`目录中。

### 2.2 Zygote进程启动过程介绍

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/zygotestartumlsequence.png" alt="Zygote进程启动过程的时序图 " style="zoom:40%;" />

```c++
// @frameworks/base/cmds/app_process/app_main.cpp
int main(int argc, char* const argv[])
{
    ......
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) { // 1
            // 如果当前运行在zygote进程中，则将zygote设置为true
            zygote = true; // 2
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) { // 3
            // 如果当前运行在SystemServer进程中，则将startSystemServer设置为true
            startSystemServer = true; // 4
        } 
        ...
    }
    ......
    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }
    // 如果运行在Zygote进程中
    if (zygote) { // 5
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote); // 6
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

> **Zygote进程都是通过fock自身来创建子进程的，这样Zygote进程以及它的子进程都可以进入app_main.cpp的main函数**。所以main函数中需要区分当前运行在哪个进程。如果是Zygote进程，则执行`AppRuntime`的`start`函数。

在[AndroidRuntime.cpp](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/source_code/AndroidRuntime.cpp.md)源码中，

* 创建Java虚拟机

* 为Java虚拟机注册JNI方法

* 通过app_main的main函数传递过来的参数，找到ZygoteInit的main方法

  > `ZygoteInit`的`main`方法是由Java语言编写的，当前运行逻辑在Native中，这就需要通过JNI来调用Java了。这样，就从**Zygote进入了Java框架层**。

ZygoteInit的[main](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/source_code/ZygoteInit.md)方法中主要做了4件事儿：

* 创建一个Server端的Socket。（名为"zygote"的Socket，用于等待AMS请求Zygote创建新的应用进程）

  > @ZygoteServer.java的`registerServerSocket`方法。
  >
  > 名称拼接规则：`ANDROID_SOCKET_PREFIX` + `socketName`；
  >
  > 实际值为：`ANDROID_SOCKET_zygote`。

* 预加载类和资源。

* 启动SystemServer进程。

  > @ZygoteInit.java的`startSystemServer`方法

* 等待AMS请求创建新的应用方程序进程。

  > @ZygoteServer.java的[`runSelectLoop`](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/source_code/Method_runSelectLoop.md)方法

### 2.3 Zygote进程启动总结

* (1) 创建`AppRuntime`并调用其`start`方法，启动Zygote进程。
* (2) 创建Java虚拟机并为Java虚拟机注册JNI方法
* (3) 通过JNI调用`ZygoteInit`的`main`函数进入Zygote的Java框架层
* (4) 通过`registerZygoteSocket`方法创建服务器端的Socket，并通过`runSelectLoop`方法等待AMS的请求创建新的应用进程。
* (5) 启动`SystemServer`进程。

***

### 3 SystemServer处理过程

　　SystemServer进程主要用于创建系统服务，如AMS、WMS、PMS。

### 3.1 Zygote处理SystemServer进程

![Zygote处理SystemServer进程的时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/zygotedealSystemServer.png)

* 在`ZygoteInit.java`的`startSystemServer`方法中启动了SystemServer进程。
* `ZygoteInit.java`的`zygoteInit`方法中
  * `ZygoteInit.nativeZygoteInit()`用来启动Binder线程池，保证与其他进程通信。
  * 进入到`SystemServer`的`main`方法。

* 进入`SystemServer`的`main`方法

  * `RuntimeInit`的`applicationInit`方法中，调用了`invokeStaticMain`方法

  * ```java
    @frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
    private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)throws Zygote.MethodAndArgsCaller {
        Class<?> cl;
        try {
            // 通过反射得到SystemServer类
            cl = Class.forName(className, true, classLoader); // 1
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className, ex);
        }
        Method m;
        try {
            // 找到SystemServer的main方法
            m = cl.getMethod("main", new Class[] { String[].class }); // 2
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException("Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException("Problem getting static main on " + className, ex);
        }
        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException("Main method is not public and static on " + className);
        }
        throw new Zygote.MethodAndArgsCaller(m, argv); // 3
    }
    ```

    * 注释1处`className`为`com.android.server.SystemServer`，通过反射得到`SystemServer`类。
    * 注释2处找到`SystemServer`中的`main`方法。
    * 注释3处将找到的main方法传入`MethodAndArgsCaller`异常并抛出该异常。
      * 捕获该异常的代码在`ZygoteInit.java`的`main`方法中，这个`main`方法会调用SystemServer的main方法。
      * **为什么不直接调用而选择抛出异常**？因为这种抛出异常的处理会清除所有的设置过程需要的堆栈帧，并让SystemServer的mian方法看起来像是SystemServer进程的入口方法。

### 3.2 解析SystemServer进程

`SystemServer`的`main`方法中，只调用了[`run`](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/source_code/SystemServer.java.md)方法。`run`方法中：

* 加载动态库`libandroid_servers.so`。
* 创建`SystemServiceManager`，用于对系统服务进行创建、启动和生命周期管理。
* 启动引导服务、核心服务、其他服务等。这些服务的父类都是`SystemService`。([表：部分系统服务及其作用](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/table_systemservice.md))

### 3.3 SystemServer进程总结

SystemServer进程被创建后，主要做了如下工作：

* 启动Bindre线程池，这样就可以与其他进程进行通信
* 创建`SystemServiceManager`，用于对系统服务进行创建、启动和生命周期管理。
* 启动各种系统服务。

***

### 4 Launcher启动过程

　　Launcher在启动过程中会请求PackageManagerService返回系统中已经安装的应用程序的信息，并将这些信息封装成一个快捷图标列表显示在系统屏幕上。Launcher作用主要是以下两点：

* (1) 作为Android系统的启动器，用于启动应用程序
* (2) 作为Android系统的桌面，用于显示和管理应用程序的快捷图标或者其他桌面组件。

### 4.1 Launcher启动过程介绍

　　SystemServer进程启动过程中会启动PackageManagerService，PMS启动后会将系统中的应用安装完成。

![Launcher启动过程时序图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/launcherstartprocess.png)

启动Launcher的入口是AMS的`systemReady`方法，经过一系列调用后，回到了AMS的`startHomeActivityLocked`方法中。实现代码如下：

```java
boolean startHomeActivityLocked(int userId, String reason) {
    //判断系统的运行模式和mTopAction的值
    if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
            && mTopAction == null) { // 1
        return false;
    }
    // 获取Luncher启动所需要的Intent
    Intent intent = getHomeIntent(); // 2
    ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
    if (aInfo != null) {
        intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        aInfo = new ActivityInfo(aInfo);
        aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
        ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                aInfo.applicationInfo.uid, true);
        if (app == null || app.instr == null) { // 3
            intent.setFlags(intent.getFlags() | Intent.FLAG_ACTIVITY_NEW_TASK);
            final int resolvedUserId = UserHandle.getUserId(aInfo.applicationInfo.uid);
            final String myReason = reason + ":" + userId + ":" + resolvedUserId;
            //启动Luncher
            mActivityStarter.startHomeActivityLocked(intent, aInfo, myReason); // 4
        }
    } else {
        Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
    }
    return true;
}

Intent getHomeIntent() {
    Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
    intent.setComponent(mTopComponent);
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
        intent.addCategory(Intent.CATEGORY_HOME);
    }
    return intent;
}
```

* `mFactoryTest`代表系统运行模式，系统运行模式有三种：**非工厂模式**、**低级工厂模式**、**高级工厂模式**。
* `mTopAction`用来描述第一个被启动的Activity组件的action，默认值为`Intent.ACTION_MAIN`。
* 当`mFactoryTest`为低级工厂模式并且`mTopAction==null`，直接返回false。
* `getHomeIntent`方法中创建Intent，非低级工厂模式下，将Category设置为`Intent.CATEGORY_HOME`。
* 注释3处判断符合Action为`Intent.ACTION_MAIN`、Category为`Intent.CATEGORY_HOME`的应用是否已经启动
* 如果还未启动，调用注释4处的方法启动。这个被启动的应用就是**Launcher**，因为Launcher的的清单文件中匹配了Action和Category这两个值。

### 4.2 Launcher中应用图标显示过程

* `Launcher.java`的`onCreate`方法

* 调用`LauncherAppState.java`的`setLauncher`方法，并将Launcher对象传入方法中。

* 调用`LauncherModel.java`的`initialize`方法，并将传入的Launcher封装成一个弱引用对象`callbacks`。

* 回到`Launcher.java`的`onCreate`方法中，调用了`LauncherModel.java`的`startLoader`方法，在该方法中：

  * 创建HandlerThread对象
  * 创建Handler，并且传入HandlerThread的Looper。Handler用于向HandlerThread发送消息。
  * 创建LoaderTask
  * 将LoaderTask作为消息发送给HandlerThread。

* `LoaderTask`实现了Runnable接口，并且是`LauncherModel`的内部类，当LoaderTask所描述的消息被处理时，会调用它的`run`方法。

* 在`LoaderTask`的`run`方法中，

  * 加载工作区信息
  * 绑定工作区信息
  * 加载系统已经安装的应用程序信息

  > **Launcher是用工作区的形式来显示系统安装的应用程序的快捷图标的，每一个工作区都是用来描述一个抽象桌面的，它是由n个屏幕组成，每个屏幕又分为n个单元格，每个单元格用来显示一个应用程序的快捷图标**。

* 加载系统已经安装的应用程序是调用`LauncherModel`的`loadAllApps`方法

* 然后调用回到了`Launcher`的`bindAllApplications`方法

* 调用`AllAppsContainerView` 的`setApps`方法，并将包含应用信息的列表apps传进去。

  > 应用信息列表apps是一个`List< Applnfo>`对象。

* 然后setApps方法中又将包含应用信息的列表apps传递给了`AlphabeticalAppsList`的`setApps`方法中。

* `AllAppsContainerView`中的`onFinishInflate`方法，会在`AllAppsContainerView`加载完XML布局时调用。

  ```java
  @Override
  protected void onFinishInflate() {
      super.onFinishInflate();
      ...
      mAppsRecyclerView = (AllAppsRecyclerView) findViewById(R.id.apps_list_view); // 1
      mAppsRecyclerView.setApps(mApps); // 2
      mAppsRecyclerView.setLayoutManager(mLayoutManager); // 3
      mAppsRecyclerView.setAdapter(mAdapter);
      ...
  }
  ```

  * 注释1处得到的`AllAppsRecyclerView`用来显示App列表。
  * 注释2处将此前`AlphabeticalAppsList`的对象`mApps`设置进去。
  * 注释3处为`AllAppsRecyclerView`设置Adapter。

* 至此，应用程序快捷图标的列表就都显示在屏幕上了。

