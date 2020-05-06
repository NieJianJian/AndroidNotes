## Glide源码分析

### 1. Glide的生命周期

with方法传入一个Context参数：

* 如果with方法传入的是Application，会通过调用getApplicationManager()来获取一个RequestManager对象，不需要处理生命周期，因为Application对象的生命周期就是应用程序的生命周期。
* 传入的不是Application，就会像当前Activity添加一个隐藏的Fragment。创建隐藏Fragment的目的是知道加载的生命的周期，因为Fragment的生命周期和Activity是同步的，如果Activity被销毁了，Fragment是可以监听到的，这样Glide就可以捕获这个事件停止图片加载。
* 如果是在非线程中调用，不管传入的是什么，都按照Application处理。

