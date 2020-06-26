## 内存泄露

　　内存泄露简单来说就是该释放的对象没能释放，一直被某个或某些实例所持有却不再被使用导致 GC 不能正常回收。GC只会回收那些不可达的对象，如果这些本该被销毁的被错误的持有，那么就造成了内存泄露。内存泄露的对象持续增加的话，应用可用的内存空间就会越来越小，GC的操作也就会频繁触发，也会使得App变慢，降低UI的流畅度，最终可能会升级为内存溢出，导致Crash。

具体内存的分配和回收可以参考[Java内存区域与内存溢出异常](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book1/Chapter2.md)和[垃圾收集器与内存分配策略](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book1/Chapter3.md)。

***

### 内存泄露的场景

#### 1. 静态变量引起的内存泄露

先来看下面几个问题：

* **Q**：静态变量什么时候加载？

  **A**：是在第一次类加载的时候分配内存的，并且存在于方法区。

* **Q**：类什么时候加载？

  **A**：当我们启动一个APP的时候，系统会创建一个进程，并且会加载一个自己的VM实例，然后代码运行在VM上，类的加载、卸载和回收都由VM管理。**有且仅有5种情况**需要对一个类进行加载（[参考链接](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book1/Chapter7.md)），就是在类第一次加载的时候，初始化了静态变量。

* **Q**：类什么时候卸载？

  **A**：一般情况，类都是由ClassLoader加载的，并且没有提供类卸载的方法，只要ClassLoader存在，类就不会被卸载，而默认的ClassLoader的生命周期和进程一致。

```java
public class MainActivity extends Activity {
    private static Context sContext;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        sContext = this;
    }
}
```

上面的代码中，Activity无法正常销毁，因为静态变量`sContexgt`引用了它。只要是静态变量直接持有或者间接持有的实例，都无法销毁。

**使用单例模式的时候，如果要传入Context对象，注意是传入Application还是Activity**。

#### 2. 非静态内部类引起的

外部类调用非静态内部类的时候，会将一个自身的实例，通过非静态内部类构造方法的参数的形式传入，这样非静态内部类就持有了外部类的引用。如果非静态内部类中还有代码没有执行完成的话，外部类是无法销毁的，这样就造成了内存泄露。还有就是非静态内部类中创建了一个静态变量，并且将外部类的实例赋值给它的话，那就外部类的就无法正常回收了。

我们常见的一个非静态内部类引起的内存泄露，就是——Handler。

```java
Handler mHandler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
    }
};
```

上述是我们常用来声明Handler的一种方式。首先代码编写过程，就会给出警告，不建议这么写。

它导致内存泄露的引用链是：

* Handler的声明方式是匿名内部类，首先，它会持有外部Activity的引用
* Handler发送消息后，会将封装的Message发送给Looper，Message会持有Handler引用。
* 主线程的消息循环Looper和应用的生命周期一样长。
* 如果消息还未执行，Activity想要销毁，是无法正常回收的。

系统也提示了改善的办法：

```java
class MyHandler extends Handler {
    private final WeakReference<MainActivity> mReference;
    public MyHandler(MainActivity activity) {
        mReference = new WeakReference<MainActivity>(activity);
    }
    @Override
    public void handleMessage(Message msg) {
        MainActivity activity = mReference.get();
        if (activity != null) {
            ...
        }
    }
}
```

* 将Handler声明为静态的，这样就不会持有外部的引用了
* 通过弱引用的方式来引用Activity。

#### 3. 监听器引起的内存泄露

在java 编程中，我们都需要和监听器打交道，通常一个应用当中会用到很多监听器，我们会调用一个控件的诸如addXXXListener()等方法来增加监听器，但往往在释放对象的时候却没有记住去删除这些监听器，从而增加了内存泄漏的机会。

#### 4. 各种连接引起的内存泄露

比如数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，除非其显式的调用了其close（）方法将其连接关闭，否则是不会自动被GC 回收的。

#### 5. 动画引起的内存泄露

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    Button button = findViewById(R.id.button1);
    ObjectAnimator animator = ObjectAnimator.ofFloat(button,
            "rotation", 0, 360).setDuration(2000);
    animator.setRepeatCount(ValueAnimator.INFINITE);
    animator.start();
    // animator.cancel();
}
```

如果属性动画一直无限循环，在Activity退出的时候没有在onDestroy中停止动画，由于动画持有了button，button又持有了Activity的实例，所以会导致内存泄露。

#### 6. WebView导致的内存泄露

造成内存泄露的原因：[WebView造成的内存泄露](https://blog.csdn.net/anhenzhufeng/article/details/80312134)。

***

### Memory Profiler

1. [使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.google.cn/studio/profile/memory-profiler?hl=zh_cn)

2. [Profiler入门系列（一）——内存分析](https://www.jianshu.com/p/ff2d7c5f043b)

3. [使用 Android Studio Profiler 工具解析应用的内存和 CPU 使用数据](https://mp.weixin.qq.com/s/UNhj37vvu-Ake7hIAB6VSw)

### LeakCanary

LeakCanary的作用时自动追踪内存泄露问题。原来的MAT分析内存泄露的核心步骤是：

* 发生OOM或做一些可能存在内存泄露的操作后，导出HPROF文件。
* 利用MAT结合代码分析，来发现一些引用异常，比如哪些对象本来应该被回收，却还在系统堆中，那么它就是内存泄露。

LeakCanary就是能自动完成内存追踪、检测、输出结果的工具。

* 通过这种形式实现Activity内存泄露检测的原理是在API 14（Android 4.0）时增加了ActivityLifecycleCallbacks，通过这个callback就可以监控Activity的生命周期。
* 默认情况下LeakCanary会监控所有Activity的生命周期，并且在Activity的onDestroy函数之后将该Activity添加到内存泄露监控队列，也就是在RefWatcher.watch()中创建一个KeyedWeakReference到被监控的对象。
* 接下来，在后台线程中检测这个引用是否被清除，如果没有将会触发GC。
* 如果引用仍然没有清除，将heap内存dump到一个.hprof文件并存放到手机系统里。
* HeapAnalyzerService在另外一个独立的进程中启动，使用HeapAnalyzer解析heap内存通过HAHA这个项目HeapAnalyzer计算出到GC ROOTS的最短引用路径来决定是否发生Leak，然后建立导致泄露的引用链。
* 结果被回传到应用程序进程的DisplayLeakService中，然后输出log并显示一个泄露的消息通知。
* Android 4.0以下没有ActivityLifecycleCallbacks，可以手动添加Activity监控，即在Activity的onDestroy函数中调用RefWatcher的watch函数。

[LeakCanary原理解析](https://www.jianshu.com/p/261e70f3083f)