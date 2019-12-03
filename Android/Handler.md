### Android中的消息机制

#### 1.处理消息的手段——Handler、Looper与Message

​		我们知道Android应用在启动时，会默认有一个主线程(UI线程)，在这个线程中会关联一个消息队列，所有的操作都会被封装成消息然后交给主线程来处理。为了保证主线程不会主动退出，会将获取消息的操作放在一个死循环中，这样程序就相当于一直在执行死循环，因此不会退出。

​		UI线程的消息循环是在ActivityThread.main方法中创建的，该函数为Android应用程序的入。源代码如下：

```java
public static void main(String[] args){
    // ......
    Process.setArgsV0("<pre-initialized>");
    Looper.prepareMainLooper(); // 1.创建消息循环Looper
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if (sMainThreadHandler == null){
        sMainThreadHandler = thread.getHandler(); // UI线程的Handler
    }
    AsyncTask.init();
    // ......
    Looper.loop(); // 2.执行消息循环
    throw new RuntimeException("Main thread loop uexpectedly exited");    
}
```

​		执行ActivityThread.main 方法后，应用程序就启动了，并且会一直从消息队列中取消息，然后处理消息，使得系统运转起来。那么系统是如何将消息投递到消息队列中的？又是如何从消息队列中获取消息并且处理消息的呢？答案就是Handler。

​		在子线程中执行完耗时操作后，很多情况下需要更新UI，但我们都知道，不能在子线程中更新UI。此时最常用的手段就是通过Handler将一个消息post到UI线程中，然后再在Handler的handleMessage方法中进行处理。但是有一个点需要注意，那就是该Handler必须在主线程中创建，简单示例如下：

```java
class MyHandler extends Handler {
    @Override
    public void handleMessage(Message msg){
        // 更新UI
    }
}

MyHandler mHandler = new MyHandler();
// 开启新线程
new Thread() {
    public void run() {
        // 耗时操作
        mHandler.sendEmptyMessage(123);    
    }    
}
```

​		为什么必须要这么做？其实每个Handler都会关联一个消息队列，消息队列被封装在Looper中，而每个Looper又会关联一个线程（Looper通过ThreadLocal）封装，最终就等于每个消息队列会关联一个线程。Handler就是一个消息处理器，将消息投递给消息队列，然后再由对应的线程从消息队列中逐个取出消息，并且执行。默认情况下，消息队列只有一个，即主线程的消息队列，这个消息队列是在ActivityThread.main方法中创建的，通过Looper.prepareMainLooper()来创建，最后执行Looper.loop()来启动消息循环。那么Handler是如何关联消息队列以及线程的呢？我们看看如下源代码：

```java
public Handler() {
    // ......
    mLooper = Looper.myLooper(); // 获取Looper
    if (mLooper == null) {
        throw new RuntimeExcetion(
                "Can't create handler inside thread that has not called Looper.prepare()");
    }    
    mQueue = mLooper.mQueue; // 获取消息队列
    mCallback = null;    
}
```

​		从Handler默认的构造函数中可以看到，Handler会在内部通过Looper.getLooper()来获取Looper对象，并且与之关联，最重要的就是消息队列。那么Looper.getLooper()又是如何工作的呢？我们继续往下看：

```java
public static Looper myLooper() {
    return sThreadLocal.get();
}

// 设置UI线程的Looper
public static void prepareMainLooper() {
    prepare();
    setMainLooper(myLooper());
    myLooper().mQueue.mQuitAllowed = false;
}

private synchronized static void setMainLooper(Looper looper) {
    sMainLooper = looper;    
}

// 为当前线程设置一个Looper
public static void prepare() {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper());
}
```

​		从上述程序中我们看到myLooper()方法是通过sThreadLocal.get()来获取的。那么Looper对象又是什么时候存储在sThreadLocal中的呢？眼见的朋友可能看到了，上面贴出的代码中给出了一个熟悉的方法——prepareMainLooper()，在这个方法中调用prepare()方法，在这个prepare方法中创建了一个Looper对象，并且将该对象设置给了sThreaLocal。这样，队列就与线程关联上了！这样以来，不同的线程就不能访问对方的消息队列。

​		再回到Handler中来，消息队列通过Looper与线程关联上，而Handler又与Looper关联，因此，Handler最终就和线程、线程的消息队列关联上了。这样就能解释上面提到的问题，"为什么要更新UI的Handler必须要在主线程中创建？"。就是因为Handler要与主线程的消息队列关联上，这样handleMessage才会执行在UI线程，此时更新UI才是线程安全的！

​		创建了Looper后，如何执行消息循环呢？通过Handler来post消息给消息队列（链表），那么消息是如何被处理的呢？答案就是在消息循环中，消息循环的建立就是通过Looper.loop()方法。源代码如下：

```java
public static void loop() {
    Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper;Looper.prepare wasn't called on this thread.");
    }
    MessageQueue queue = me.mQueue; // 1.获取消息队列
    // ......
    while (true) { // 2.死循环，即消息循环
        Message msg = queue.next(); // 3.获取消息（might block）
        if (msg != null) {
            if (msg.target == null) {
                // No target is a magic identifier for the quit message。
                return;
            }
            // ......
            msg.target.dispatchMessage(msg); // 4.处理消息
            // ......
            msg.recycle();
        }
    }
}
```

​		从上述程序中可以看到，loop方法中实质上就是建立一个死循环，然后通过从消息队列中逐个取出消息，最后处理消息的过程。对于Looper我们总结一下：通过Looper.prepare()来创建Looper对象（消息队列封装在Looper对象中），并且保存在sThreadLocal中，然后通过Looper.loop()来执行消息循环，这两步通常是成对出现的。

​		最后，我们看看消息处理机制，我们看到上面的代码中第4步通过msg.target.dispatchMessage(msg)来处理消息。其中msg是Message类型，我们看源代码：

```java
public final class Message implements Parcelable {
    Handler target; // target处理
    Runnable callback; // Runnable类型的callback
    Message next; // 下一条消息，消息队列是链式存储的
    // ......
}
```

​		从源代码中可以看到，target是Handler类型。实际上就是转了一圈，通过Handler将消息投递给消息队列，消息队列又将消息分发给Handler来处理。继续看源代码：

```java
// 消息处理函数，子类覆盖
public void handleMessage (Message msg) {
}

private final void handleCallback(Message message){
    message.callback.run();
}

// 分发消息
public void dispatchMessage (Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

​		从上述程序中可以看到，dispatchMessage只是一个分发的方法，如果Runnable类型的callback为空，则执行handleMessage来处理消息，该方法为空，我们会将更新UI的代码写在该函数中；如果callback不为空，则执行handleCallback来处理，该方法会调用callback的run方法。其实这是Handler分发的两种类型，比如我们post(Runnable callback)则callback就不为空，当我们使用Handler来sendMessage时通常不会设置callback，因此，也就执行handlerMessage这个分支。我们看看两种实现：

```java
public final boolean post(Runnable r){
    return sendMessageDelayed(getPostMessage(r), 0);
}

private final Message getPostMessage (Runnbale r){
    Message m = Message.obtain();
    m.callback = r;
    return m;
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long updateMillis) {
    boolean sent = false;
    MessageQueue queue = mQueue;
    if (queue != null) {
        msg.target = this; // 设置消息的target为当前的Handler对象
        sent = queue.enqueueMessage(msg, uptimeMillis); // 将消息插入到消息队列
    } else {
        // ......
    }
    return sent;
}
```

​		从上述程序中可以看到，在post(Runnable r) 时，会将Runnable包装成Message对象，并且将Runnable对象设置给Message对象的callback字段，最后会将该Message对象插入消息队列。sendMessage也是类似实现。

```java
public final boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
}
```

​		不管时post一个Runnable还是Message，都会调用sendMessage(msg, time)方法。Handler最终将消息追加到MessageQueue中，而Looper不断地从MessageQueue中读取消息，并且调用Handler的dispatchMessage方法，这样消息就能源源不断地被产生、添加到MessageQueue、被Handler处理。



#### 2.在子线程中创建Handler为何会抛出异常

​		首先看代码：

```java
new Thread() {
    Handler handler = null;
    public void run(){
        handler = new Handler();    
    }
}.start();
```

​		前面说过，Looper对象是ThreadLocal地，即每个线程都有自己地Looper，这个Looper可以为空。但是，当你要在子线程中创建Handler对象时，如果Looper为空，那么就会抛出"Can't create handler inside thread that has not called Looper.prepare()"异常。为什么会这样？看源代码：

```java
public Handler {
    // ......
    mLooper = Looper.myLooper(); // 获取myLooper
    // 抛出异常
    if (mLooper == null) {
        throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = null;
}
```

​		从上述程序中我们看到，当mLooper对象为空时，抛出该异常。这是因为该线程中的Looper对象还没创建，因此sThreadLocal.get()会返回null。Handler的原理就是要与MessageQueue建立关联，并且将消息投递给MessageQueue，如果连MessageQueue都没有，那么mHandler就没有存在的必要，而MessageQueue又被封装在Looper中，因此，创建Handler时Looper一定不能为空，解决办法如下：

```java
new Thread() {
    Handler handler = null;
    public void run(){
        Looper.prepare(); // 1.为当前线程创建Looper，并且会绑定到ThreadLocal中
        handler = new Handler();
        Looper.loop(); // 2.启动消息循环
    }
}.start();
```













