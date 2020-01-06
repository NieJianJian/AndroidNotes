## 第2章 Android系统启动

### Android系统启动流程

流程图如下：

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/androidsystemloader.png" alt="Android系统启动流程图 " style="zoom:60%;" />

* **1.启动电源以及系统启动**

  当按下电源时引导芯片代码从预定义的地方（固化在ROM）开始执行。加载引导程序BootLoader到RAM，然后执行。

* **2.引导程序BootLoader**

  引导程序BootLoader是在Android操作系统开始运行前的一个小程序，它的主要作用是把系统OS拉起来并运行。

* **3.Linux内核启动**

  当内核启动时，设置缓存、被保护存储器、计划列表、加载驱动。当内核完成系统设置时，它首先在系统文件中寻找**`init.rc`**文件，并**启动init进程**。

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