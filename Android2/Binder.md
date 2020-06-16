## Binder

### 1. 简介

Binder实现了IBinder接口。

* 从IPC角度，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是`/dev/binder`，该通信方式Linux中没有。
* 从Android Frameworks角度，Binder是ServieManager连接各种Manger（ActivityManager、WindowManager等）和相应的ManagerService的桥梁。
* 从Android应用层来说，Binder是客户端与服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端的服务或数据。

通过[AIDL](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/AIDL.md)可以连接到Binder的工作机制，不过要注意两点：

* 当客户端发起远程请求时，由于当前线程会被挂起直至服务端进程返回数据，所以如果一个远程方法是很耗时的，那么不能在UI线程中发起此远程请求。
* 由于服务端的Binder方法运行在Binder线程池中，所以Binder方法不管是否耗时都应该采用同步的方式去处理。

Binder的工作机制图如下：

![图：Binder的工作机制](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/binderworkingjizhi.jpeg)

ADIL文件是实现Binder的一种方式，也就是说，AIDL并不是实现Binder的必需品。

**Binder意外死亡怎么办**：Binder运行在服务端进程，如果服务端进程由于某种原因异常终止，这时候BInder连接就断裂了（称之为Binder死亡）。关键是我们不知道Binder连接已经断了，客户端就会受到影响。为了解决这个问题，Binder提供了两个重要方法`linkToDeath`和`unlinkToDeath`。

* 通过`linkToDeath`给Binder设置一个死亡代理，当Binder死亡时，我们会收到通知，然后重新发起连接请求从而回复连接。

* 首先声明DeathRecipient对象，当Binder死亡时，系统会回调binderDied方法，我们就可以移除之前绑定的Binder代理并重新绑定远程服务：

  ```java
  private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
      @Override
      public void binderDied() {
          if (mBookManager == null)
              return;
          mBookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
          mBookManager = null;
          // 这里重新绑定远程service
      }
  };
  ```

* 然后在客户端绑定远程服务成功后，给binder设置死亡代理

  ```java
  mService = IMessageBoxManager.Stub.adInterface(binder);
  binder.linkToDeath(mDeathRecipient, 0);
  ```

***

### 参考文章

* [Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)