## Handler的原理

### 1.处理消息的手段——Handler、Looper与Message

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

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this; // 设置消息的target为当前的Handler对象
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis); // 将消息插入到消息队列
}
```

​		从上述程序中可以看到，在post(Runnable r) 时，会将Runnable包装成Message对象，并且将Runnable对象设置给Message对象的callback字段，最后会将该Message对象插入消息队列。sendMessage也是类似实现。

```java
public final boolean sendMessage(Message msg) {
    return sendMessageDelayed(msg, 0);
}
```

​		不管是post一个Runnable还是send一个Message，都会调用sendMessageAtTime(msg, time)方法。Handler最终将消息追加到MessageQueue中，而Looper不断地从MessageQueue中读取消息，并且调用Handler的dispatchMessage方法，这样消息就能源源不断地被产生、添加到MessageQueue、被Handler处理。

### 2. MessageQueue如何处理消息

我们添加消息，最终会调用到MessageQueue的enqueueMessage方法：

```java
boolean enqueueMessage(Message msg, long when) {
    ...
    synchronized (this) {
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 插入到queue
        }
        if (needWake) { nativeWake(mPtr); }
    }
    return true;
}
```

上述代码可以看出，入队的Message，如果满足条件，则立即执行，如果不满足，则加入队列中。

处理消息则是调用MessageQueue的next方法取出消息进行处理：

```java
Message next() {
    ...
    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis); // 1
        synchronized (this) {
            Message msg = mMessages;
            if (msg != null) { 
                if (now < msg.when) {
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, 
                            Integer.MAX_VALUE); // 2
                } else {
                    ...
                    return msg;
                }
            } else {
                nextPollTimeoutMillis = -1; // 3
            }
    ...
}
```

上述代码可以看出，next也是通过死循环去取消息，

* 先看注释2处，如果msg不为null，并且还不到执行的时间，则设置nextPollTimeoutMillis的值；
* 如果msg不为null，并且达到了执行的条件，则直接返回msg去执行；
* 如果msg为null，则将nextPollTimeoutMillis置为-1；

接着我们看nativePoolOnce，会调用底层的NativeMessageQueue，假设该方法中发现nextPollTimeoutMillis值为5000，则阻塞5000ms后自动返回，如果发现nextPollTimeoutMillis值为-1，说明没有消息需要处理，则会一直阻塞，那什么时候唤醒呢？是在MessageQueue的enqueueMessage方法中，最后会判断，如果neekWake为true，就会调用底层的nativeWake方法，就会唤醒阻塞，也就是nativePoolOnce不再阻塞，返回后继续往下执行代码。底层采用的是epoll机制，用来监控描述符，如果描述符就绪，则会通知相应的程序进行读写。

### 3. 消息空闲IdleHandler

IdleHanlder是MessageQueue的内部类

```java
/**
 * Callback interface for discovering when a thread is going to block
 * waiting for more messages.
 */
public static interface IdleHandler {
    /**
     * Called when the message queue has run out of messages and will now
     * wait for more.  Return true to keep your idle handler active, false
     * to have it removed.  This may be called if there are still messages
     * pending in the queue, but they are all scheduled to be dispatched
     * after the current time.
     */
    boolean queueIdle();
}
```

使用方式入如下：

```java
MessageQueue.IdleHandler idleHandler = new MessageQueue.IdleHandler() {
        @Override
        public boolean queueIdle() {
            ...
            return false; 
        }
    };
Looper.myQueue().addIdleHandler(idleHandler);
```

在IdleHanlder的queueIdle()方法中，返回false表示一次性消费，执行完就移除掉；返回true表示这次执行完，下一次还会执行。

我们来看IdleHanlder的相关方法和内容：

```java
public final class MessageQueue {
    ...
    private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
    private IdleHandler[] mPendingIdleHandlers;

    public void addIdleHandler(@NonNull IdleHandler handler) {
        if (handler == null) {
            throw new NullPointerException("Can't add a null IdleHandler");
        }
        synchronized (this) {
            mIdleHandlers.add(handler);
        }
    }

    public void removeIdleHandler(@NonNull IdleHandler handler) {
        synchronized (this) {
            mIdleHandlers.remove(handler);
        }
    }
```

具体的调用还是在MessageQueue的next方法中：

```java
Message next() {
    ...
    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis); // 1
        synchronized (this) {
            Message msg = mMessages;
            if (msg != null) { 
                if (now < msg.when) {
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, 
                            Integer.MAX_VALUE); // 2
                } else {
                    ...
                    return msg;
                }
            } else {
                nextPollTimeoutMillis = -1; // 3
            }
            // 队列空闲时，执行idle队列
            if (pendingIdleHandlerCount < 0&& (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                mBlocked = true;
                continue; // 空闲队列为null，则继续执行for循环
            }
            if (mPendingIdleHandlers == null) {
               mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        } 
        // 运行空闲队列
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler
            boolean keep = false;
            try {
                keep = idler.queueIdle(); // 4
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {  // 5
                synchronized (this) {
                    mIdleHandlers.remove(idler); // 6
                }
            }
        }
        // 将空闲队列计数置为0，这样就不用再运行它们了。
        pendingIdleHandlerCount = 0;
        // 调用空闲队列时，可能已经传递了一个新消息，所以返回重新执行一遍for循环
        nextPollTimeoutMillis = 0;
    } 
}
```

我们来总结一下空闲队列的处理过程：

* 如果next方法查询到msg，则会直接return，去执行消息；
* 如果当前队列为null，或者有消息还没到执行的时间，就会继续往下执行，运行到idle相关代码；
* 判断存放idle的集合大小是否为0，如果是则continue继续执行for循环；
* 如果idle集合不为null，则idle集合转换成数组去遍历执行；
* 每执行一个idleHandler任务，都会对queueIdle的返回值进行判断，如果是false，就将此IdleHandler任务从idle集合中移除；
* 遍历执行完成后，将idle队列计数置为0 ，这样就不用在运行它们了；
* 还要将nextPollTimeoutMillis置为0，因为执行空闲队列期间，可能有新的的Message创建，所以重新执行for循环去检查新消息。

### 4. 在子线程中创建Handler为何会抛出异常

​		首先看代码：

```java
new Thread() {
    Handler handler = null;
    public void run(){
        handler = new Handler();    
    }
}.start();
```

​		前面说过，Looper对象是ThreadLocal的，即每个线程都可以有自己的Looper，默认创建线程是没有Looper的，这个Looper可以为空。但是，当你要在子线程中创建Handler对象时，如果Looper为空，那么就会抛出"Can't create handler inside thread that has not called Looper.prepare()"异常。为什么会这样？看源代码：

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

### 5. 消息屏障

[Android 消息屏障与异步消息](https://www.jianshu.com/p/ea4a683e717e)

[Android Handler 机制（四）：屏障消息（同步屏障）](https://www.cnblogs.com/renhui/p/12875589.html)

***

***

***

**问题1：为什么主线程的Looper.loop()死循环不会导致ANR**？

　　主线程的死循环一直运行是不是特别消耗CPU资源呢？ 其实不然，这里就涉及到Linux `pipe/epoll`机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里，详情见[Android消息机制1-Handler(Java层)](https://link.zhihu.com/?target=http%3A//www.yuanhh.com/2015/12/26/handler-message-framework/%23next)，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的`epoll`机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

参考链接：[Android中为什么主线程不会因为Looper.loop()里的死循环卡死？](https://www.zhihu.com/question/34652589)

***

**问题2：如何在Handler处理每条消息开始和结束都打印一条Log**？

* 使用Looper的一个方法

  ```java
  private Printer mLogging;
  public void setMessageLogging(@Nullable Printer printer) {
      mLogging = printer;
  }
  ```

* 在Looper的loop()方法中，每次处理消息的时候，都会进行如下判断：

  ```java
  public static void loop() {
      ...
      for (;;) {
          ...
          // This must be in a local variable, in case a UI event sets the logger
          final Printer logging = me.mLogging;
          if (logging != null) {
              logging.println(">>>>> Dispatching to " + msg.target + " " +
                      msg.callback + ": " + msg.what);
          }
          ...
          try {
              msg.target.dispatchMessage(msg);
              dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
          } 
          ...
          if (logging != null) {
              logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
          }
          ...
      }
  }
  ```

  我们可以看到，在调用Handler的dispatchMessage方法的前后，都对`logging`进行了判断，如果不为null，则打印相应的log。

  所以我们只需要在代码中进行调用setMessageLogging方法并穿进去一个Printer对象即可。

  ActivityThread的main方法中，就有类似的调用：

  ```java
  public static void main(String[] args) {
      ...
      if (false) {
          Looper.myLooper().setMessageLogging(new LogPrinter(Log.DEBUG, "ActivityThread"));
      }
  }
  ```

***

**问题3：Handler导致内存泄露的路径**

　　Handler的声明方式一般为匿名内部类，这种情况可能会导致内存泄露。

* 首先，匿名内部类会持有外部类的引用，也就是Handler会持有调用它的外部类的引用。

* post或者send方法，最终都会调用到enqueueMessage方法中，方法中有如下一行代码

  ```java
  msg.target = this;
  ```

  设置Message的target字段为当前的Handler对象。

* 结合上面的步骤，如果Message处理消息，没有处理完成，或者设置了delay，还没到执行的时间，外部的Activity销毁了，由于Handler持有Activity的引用，Message持有Handler的引用，Message会间接的持有外部Activity的引用，Message没有处理完成，或者还没来得及执行，是不会销毁的，也导致外部Activity无法销毁，从而导致了内存泄露。

* **解决办法**

  * 关闭Activity的时候，调用Handler的removeCallbackAndMessage(null)方法，取消所有排队的Message。
  * 将Handler创建为静态内部类，静态内部类不会持有外部类的引用，并且使用WeakReference来包装外部类对象。当Activity想关闭销毁时，mHandler对它的弱引用没有影响，该销毁销毁；当mHandler通过WeakReference拿不到Activity对象时，说明Activity已经销毁了，就不用处理了，相当于丢弃了消息。

***

