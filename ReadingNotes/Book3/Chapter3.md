## 第3章 应用程序进程启动过程

### 1 应用程序进程启动过程介绍

### 1.1 AMS发送启动应用程序进程请求

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/amssendstartappprocess.png" alt="AMS发送启动应用程序进程请求过程的时序图" style="zoom:80%;" />

* AMS会调用`startProcessLocked`方法向zygote进程发送请求

  ```java
  private final void startProcessLocked(...) {
      ...
      int uid = app.uid; // 1 获取要创建的应用程序进程的用户ID
      ...
      if (!app.isolated) {
          // 2 对gids进行创建和复制
          if (ArrayList.isEmpty(permGids)) {
              gids = new int[3];
          } else {
              gids = new int[permGids.length + 3];
              System.arraycopy(permGids, 0, gids, 3, permGids.length);
          }
          gid[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
          gid[1] = UserHandle.getCacheAppGid(UserHandle.getAppId(uid));
          gid[2] = UserHandle.getUserGid(UserHandle.getUserId(uid));
      }
      ...
      if (entryPoint == null) entryPoint = "android.app.ActivityThread"; // 3
      ...
      if (hostingType.equals("webview_service")) {
          ...
      } else {
          // 4 启动应用程序进程
          startResult = Process.start(entryPoint, app.peocessName, uid, uid, gids,
                  debugFlags, mountExternal, app.info.targetSdkVersion, seInfo,
                  requiredAbi, instructionSet, app.info.dataDir,invokeWith,entryPointArgs);
      }
  }
  ```

  * 注释1处得到创建应用程序进程的用户ID
  * 注释2处对用户组id（gids）进程创建和赋值。
  * 注释3处，如果`entryPoint`为null，则赋值为`android.app.ActivityThread`，这是应用程序进程主线程的类名。
  * 调用`Process`的`start`方法。

* 然后调用`ZygoteProcess`的`start`方法，ZygoteProcess用于保持与Zygote进程的通信状态。

* 调用`ZygoteProcess`的`startViaZygote`方法，该方法中创建字符串列表`argsForZygote`，并将应用进程的启动参数保存进去。此参数传递到下一个方法调用中。

* 调用`ZygoteProcess`的`zygoteSendArgsAndGetResult`方法，该方法的主要作用是将传入的`argsForZyogte`写入`ZygoteState`中。`ZygoteState`是`ZygoteProcess`的静态内部类，用于表示与Zygote进程通信的状态。

* `ZygoteState`是由`ZygoteProcess`的`openZygoteSocketIfNeeded`方法返回的，

  ```java
  /*主要代码*/
  primaryZygoteState = ZygoteState.connect(mSocket); // 1
  if (primaryZygoteState.matches(abi)) {return primaryZygoteStart;} // 2
  if (secondaryZygoteState == null || secondaryZygoteStaet.isClosed()) {
      secondaryZygoteState = ZygoteState.connect(mSecondarySocket); // 3
  }
  if (secondaryZygoteState.matched(abi)) {return secondaryZygoteState;}
  ```

  * 注释1处与Zygote进程建立连接，并返回`ZygoteState`类型的`primaryZygoteState`对象。
  * 注释2处判断，如果连接Zygote主模式返回的`ZygoteState`与启动应用程序进程所需的ABI不匹配，则连接Zygote辅模式。注释3处连接名为"zygote_secondary"的Socket。
  * 注释4处连接Zygote辅模式返回的ZygoteState与启动应用程序进程的所需的ABI也不匹配，则抛出`ZygoteStartFailedEx`异常。

### 1.2 Zygote接收请求并创建应用程序进程

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/zygotereceivecreateprocess.png" alt="zygote接受请求并创建应用程序进程的时序图" style="zoom:80%;" />

Socket连接成功并匹配ABI后会返回ZygoteState类型对象，ZygoteState中保存了应用进程启动参数。这样Zygote进程就收到了一个新的创建应用程序进程的请求。

* `ZygoteInit`的`main`方法中，调用了`ZygoteServer`的`runSelectLoop`方法

* `runSelectLoop`方法用来等待MAS请求创建新的应用程序进程。

  ```java
  void runSelectLoop(String abiList) throws Zygote.MethodAndArgsCaller {
      ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
      ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>(); // 1
      fds.add(mServerSocket.getFileDescriptor());
      peers.add(null);
      while (true) {
          ...
          for (int i = pollFds.length - 1; i >= 0; --i) {
              if ((pollFds[i].revents & POLLIN) == 0) {
                  continue;
              }
              if (i == 0) {
                  ZygoteConnection newPeer = acceptCommandPeer(abiList);
                  peers.add(newPeer);
                  fds.add(newPeer.getFileDesciptor());
              } else {
                  boolean done = peers.get(i).runOnce(this); // 2
                  if (done) {
                      peers.remove(i);
                      fds.remove(i);
                  }
              }
          }
      }
  }
  ```

* 结合上述方法，当AMS的请求数据到来是，实际调用的是`ZygoteConnection`的`runOnce`方法。在该方法中，

  * 调用`readArgumentList`方法获取应用程序进程的启动参数，赋值给`String[]`类型的`args`。
  * 将`args`封装到`Arguments`类型的`parsedArgs`对象中。
  * 调用`Zygote`的`forkAndSpecialize`方法来创建应用程序进程，参数为`parsedArgs`中存储的应用程序启动参数，方法返回值为`pid`。该方法主要通过fork当前进程来创建一个子进程。
  * 如果`pid==0`，这说明当前代码逻辑运行在子进程中。然后调用`handleParentProc`方法

* 再调用`ZygoteInit`的`zygoteInit`方法，该方法中创建Binder线程池。

* 调用`RuntimeInit`的`applicationInit`方法

* 调用`invokeStaticMain`方法，该方法的第一个参数`args.startClass`，其实是`android.app.ActivityThread`。

  ```java
  private static void invokeStaticMain(String className, String[] argv, ClassLoader classLoader)throws Zygote.MethodAndArgsCaller {
      Class<?> cl;
      try {
          // 通过反射得到android.app.ActivityThread类
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
      ...
      throw new Zygote.MethodAndArgsCaller(m, argv); // 3
  }
  ```

  上述代码可以看出，和Zygote处理SystemServer进程一样，通过抛出异常的处理方式清除所有的设置过程需要的堆栈帧，并让ActivityThread的main方法看起来像是应用程序进程的入口方法。

* 在`ZygoteInit`的`main`方法中，捕获到`MethodAndArgsCaller`异常时，调用`MethodAndArgsCaller`的`run`方法，`MethodAndArgsCaller`是`Zygote.java`的静态内部类。

* `MethodAndArgsCaller`的`run`方法中，

  ```java
  mMethod.invoke(null, new Object[] { mArgs });
  ```

  `mMethod`指的是`ActivityThread`的`main`方法，调用`mMethod`的`invoke`方法后，就进入了`ActivityThread`的`main`方法中了。

***

### 2 Binder线程池启动过程

* `ZygoteInit`的`zygoteInit`方法中，调用了native方法`nativeZygoteInit`来创建Binder线程池。

* `AndroidRuntime.cpp`的`JNINativeMethod`数组中，我们得知，对应`com_android_internal_os_ZygoteInit_nativeZygoteInit`方法，

* 然后调用`gCurRuntime->onZygoteInit()`。`gCurRuntime`是`AndroidRuntime`类型指针，`AppRuntime`继承自`AndroidRuntime`，`gCurRuntime`实际指向的是`AppRuntime`。

* `AppRunime`的`onZygoteInit`方法中，调用`ProcessState`的`startThreadPool`函数来启动Binder线程池。

  ```c++
  void ProcessState::startThreadPool()
  {
      AutoMutex _l(mLock);
      if (!mThreadPoolStarted) { // 1
          mThreadPoolStarted = true; // 2
          spawnPooledThread(true);
      }
  }
  ```

  支持Binder通信的进程都有一个`ProcessState`类，它里面有一个`mThreadPoolStarted`变量，用来表示BInder线程池已经启动过了，默认值为false。每次调用都要先检查这个标记，从而**确保Binder线程池只启动一次**。

* 如果Binder线程池还未启动，调用`spawnPooledThread`方法。该函数来**创建线程池中的第一个线程**。也就是**主线程**。

* Binder线程是一个`PoolThread`，调用`PoolThread`的`run`函数来启动一个新的线程。

  ```c++
  class PoolThread : public Thread
  {
  ...  
  protected:
      virtual bool threadLoop()
      {
          IPCThreadState::self()->joinThreadPool(mIsMain); // 1
          return false;
      }
      const bool mIsMain;
  };
  ```

  `PoolThread`继承自`Thread`类，注释1处调用`IPCThreadState`的`joinThreadPool`函数，将当前线程注册到Binder驱动程序中，这样我们创建的线程就加入了Binder线程池中，新创建的应用程序进程就支持Binder进程间通信了。

  我们只需要创建当前进程的Binder对象，并将它注册到ServiceManager中就可以实现Binder进程间通信。
