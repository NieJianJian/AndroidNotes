## Binder

### 1. 简介

Binder实现了IBinder接口。

* 从IPC角度，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是`/dev/binder`，该通信方式Linux中没有。
* 从Android Frameworks角度，Binder是ServieManager连接各种Manger（ActivityManager、WindowManager等）和相应的ManagerService的桥梁。
* 从Android应用层来说，Binder是客户端与服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端的服务或数据。

***

### 2. 

