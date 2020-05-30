## Android多线程实现方式

Android中多线程的实现方式主要有以下几种：

* Handler

* AsyncTask

* HandlerThread

* IntentService

* ThreadPoolExecutor

***

***

***

### 子线程更新UI的方法

[Android子线程中更新UI的方法](https://blog.csdn.net/u012975370/article/details/53928319)

***

***

***

### Handler

Handler的使用方式不多讲了，一般都会。可以看一下[Handler原理](https://github.com/NieJianJian/AndroidNotes/blob/master/Android/Handler.md)。

***

***

***

### AsyncTask

　　如果通过Thread执行耗时任务，在操作完成之后，可能需要更新UI，一般做法是通过Handler投递一个消息给UI线程。这样做代码相对臃肿，如果多个任务同事执行，不易对线程精确控制。

　　AsyncTask是轻量级的异步任务类，它封装了Thread和Handler，在线程池中执行异步任务，然后把执行进度和结果传递给主线程。特别耗时的任务，还是建议使用线程池。

#### 定义

```java
public abstract class AsyncTask<Params, Progress, Result> { }
```

3种泛型类型分别表示参数类型、后台任务执行进度类型、返回的结果类型。并不是所有类型我们都需要，如果不需要某个参数，可以设置为Void类型。

AsyncTask的核心方法和含义如下所示：

* execute(Params... params)，运行在UI线程，用于触发一个异步任务的执行。
* onPreExecute()，运行在UI线程，在execute()之后异步任务开始之前调用，用于做一些准备工作。
* doInBackground(Params... params)，运行在线程池中，用于执行耗时的异步任务，params参数表示异步任务的输入参数。此方法中可以调用publishProgress来更新任务进度，publishProgress方法会调用onProgress方法。此方法需要返回结果给onPostExecute方法。
* onProgressUpdate(Progress... values)，运行在UI线程，在调用publishProgress时，可以在该方法中直接更新进度信息到UI组件上。
* onPostExecute(Result result)，运行在UI线程，异步任务结束后，此方法会被调用，result参数是doInBackground方法的返回值。

使用的时候，需要注意几点：

* 异步任务实例必须在UI线程中创建
* execute方法必须在UI线程中调用
* 不要在程序中直接调用onPreExectute()、onPostExecute()、doInBackground()、onProgressUpdate等方法。
* 一个任务实例只能执行一次，执行第二次会抛出异常。

#### 使用方法

```java
class DownloadTask extends AsyncTask<URL, Integer, Long> {
    @Override
    protected void onPreExecute() {
        // 初始化UI，比如启动进度条
    }
    @Override
    protected Long doInBackground(URL... urls) {
        // 执行耗时操作
        // 更新进度
        publishProgress(progress);
        return result;
    }
    @Override
    protected void onProgressUpdate(Integer... values) {
        // 更新进度UI
    }
    @Override
    protected void onPostExecute(Long result) {
        // 对结果进行处理
    }
}
```

```java
new DownloadTask().execute(url1, urls, url3);
```

#### 源码分析

首先我们先来思考几个问题，后续的源码分析中，会对这些问题一一解答：

* 为什么AsyncTask实例只能运行一次？
* 为什么AsyncTask的类必须在主线程中加载。
* 执行顺序是串行还是并行？

分析AsyncTask原理，从execute方法开始

```java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }
    mStatus = Status.RUNNING;
    onPreExecute();
    mWorker.mParams = params;
    exec.execute(mFuture);
    return this;
}
```

分析executeOnExecutor方法，主要做了如下几件事儿：

* 首先，判断该AsyncTask的状态，如果不是PENDING，则抛出异常。
* 将AsyncTask的状态置为RUNNING。
* 执行onPreExecute()方法，去做一些准备操作。
* 将UI线程传进来的参数，赋值给mWorker。
* 然后将任务交给线程池进行调度，参数为FutureTask类型，构造mFuture的时候将mWorker传进去了。
* 返回自身，使得调用者可以保持一个引用。

根据上面的代码，首先来思考下面这个问题：

**Q：为什么AsyncTask实例只能运行一次**？

**A**：因为AsyncTask是有三种状态的，看如下代码：

```java
// 初始状态
private volatile Status mStatus = Status.PENDING;
public enum Status {
    // 未执行状态
    PENDING,
    // 执行中
    RUNNING,
    // 执行完成
    FINISHED,
}
public final Status getStatus() { return mStatus; }
```

上述代码看到，AsyncTask初始状态为PENDING，代表待定状态，在执行execute方法的时候，会有对状态进行判断，如果不是PENDING状态就会抛出异常。

接着继续分析`exec.execute(mFuture);`，exec参数是在execute()方法中传入到executeOnExecutor中的。

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;
    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }
    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

从SerialExecutor的实现可以分析AsyncTask的排队执行过程。

* 系统首先会把AsyncTask的Params参数封装到FutureTask对象，FutureTask是一个并发类，在这里充当了Runnable的作用（因为FutureTask最终会继承于Runnable）。
* 接着会把这个FutureTask交给SerialExecutor的execute方法处理；
* SerialExecutor的execute方法首先会被FutureTask对象插入到任务队列mTasks中；
* 如果此时没有正在活动的AsyncTask任务，就会调用SerialExecutor的scheduleNext方法执行下一个任务。
* 同时当一个AsyncTask任务执行完成后，会调用scheduleNext方法执行下一个任务，直到所以的任务执行完毕。

从上面的执行过程可以看出，之前的一个问题就有答案了：**默认情况下，AsyncTask是串行执行的**。

上述代码还可以看出，AsyncTask有两个线程池，分别是SerialExecutor和THREAD_POOL_EXECUTOR。

* SerialExecutor只负责将异步任务排队
* 真正执行任务的是THREAD_POOL_EXECUTOR。

先来了解一下THREAD_POOL_EXECUTOR：

```java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;
private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);
    public Thread newThread(Runnable r) {
        return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
    }
};
private static final BlockingQueue<Runnable> sPoolWorkQueue =
        new LinkedBlockingQueue<Runnable>(128);
public static final Executor THREAD_POOL_EXECUTOR;
static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

上面代码可以看到，是对线程池的一些初始化设置。

线程池来执行任务，会调用mFuture的run方法，由于构造mFuture的时候把mWroker传入到构造函数中，所以，最终会调用mWroker的call方法，因此，mWorker的call方法最终会在线程池中执行，mFuture和mWorker的初始化在AsyncTask的构造函数中：

```java
public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);
            }
            return result;
        }
    };
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()", e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}

private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}
```

首先，线程池执行mFuture，会调用到mFuture的run方法，然后调用到mWorker的call方法执行后台任务，当任务执行完成后，又会回调mFuture的done方法。

上述代码可以看出，调用到mWorker的call方法中：

* 首先将`mTaskInvoked`设置为true，表示当前任务已经被调用；
* 然后调用了doInBackground方法，并将结果赋值给result。
* 最后调用postResult(result)，将消息投递到UI线程。

```java
private void postResultIfNotInvoked(Result result) {
    final boolean wasTaskInvoked = mTaskInvoked.get();
    if (!wasTaskInvoked) {
        postResult(result);
    }
}

private Result postResult(Result result) {
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

上述postResult方法可以看出，是最终将结果投递到UI线程中的。

从mFuture的done方法中调用的postResultIfNotInvoked方法的实现来看，如果期间发生异常，mWroker的call方法没有执行，或者没有正确执行，就不会调用到postResult方法，最终会在mFuture的done方法中检测是否执行成功，如果成功且未调用postResult，就调用postResult函数分发结果，否则忽略该执行结果，返回null。

接下来，我们来看Handler的相关内容。

mHandler获取一般调用的getMainHandler方法，方法如下，返回的是sHandler对象。

```java
private static Handler getMainHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}
```

sHandler其实是AsyncTask内部类InternalHandler的实例：

```java
private static final int MESSAGE_POST_RESULT = 0x1;
private static final int MESSAGE_POST_PROGRESS = 0x2;
private static InternalHandler sHandler;

private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

sHandler是一个静态的Handler对象，并且构造函数中传入的是主线程的Looper对象，为了要求执行环境能切换到主线程，就需要sHandler对象必须在主线程中初始化。静态成员会在类加载的时候进行初始化，因此变相要求**AsyncTask的类必须在主线程中加载**。

sHandler中主要处理两个信息：

* 一个是之前处理结果的postResult方法中，发送的`MESSAGE_POST_RESULT`消息；
* 一个是更新进度的，发送的是`MESSAGE_POST_PROGRESS`消息。

`MESSAGE_POST_RESULT`消息的处理中，最终调用了AsyncTask的finish方法，内容如下：

```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

* 调用了onPostExecute(result)，并将结果返回
* 将AsyncTask的状态变为FINISHED。

在之前的postResult方法中，发送消息时，构建了一个AsyncTaskResult类型的消息：

```java
private static class AsyncTaskResult<Data> {
    final AsyncTask mTask;
    final Data[] mData;
    AsyncTaskResult(AsyncTask task, Data... data) {
        mTask = task;
        mData = data;
    }
}
```

上述代码可以看出，AsyncTaskResult用来封装AsyncTask实例和执行结果。

#### 拓展知识

* Android 4.1之前需要在主线程中手动调用一次AsyncTask，保证其在主线程中初始化。Android4.1以上会被系统自动完成。
* 由于执行任务的线程池是静态的，所以，一个进程中的所有任务共用THREAD_POOL_EXECUTOR线程池。
* 任务执行默认是串行的，可以调用executeOnExecutor方法并行执行。

***

***

***

### HandlerThread

HandlerThread继承了Thread，它是一个可以使用Handler的Thread。

**Q**：主线程如何向子线程发送消息？

**A**：参考子线程向主线程发送消息使用的Handler，在主线程创建的时候，创建了Looper，并启动了消息循环

```java
Looper.prepareMainLooper(); // 创建消息循环
Looper.loop(); // 启动消息循环
```

子线程通过Hanlder发送给主线程消息，也就是将消息发送到消息队列进行处理。

反之，如果想要从主线程向子线程发送消息，只需要为子线程创建消息循环，并从主线程通过Handler发送给子线程的消息队列进行处理即可。（子线程之间发送消息同理）

**Q**：子线程中只创建Handler为何会抛出异常？

```java
new Thread() {
    Handler handler = null;
    public void run(){
        handler = new Handler(); // 会报错
    }
}.start();
```

**A**：前面说过，Looper对象是ThreadLocal的，即每个线程都可以有自己的Looper，默认创建线程是没有Looper的，这个Looper可以为空。但是，当你要在子线程中创建Handler对象时，如果Looper为空，那么就会抛出"Can't create handler inside thread that has not called Looper.prepare()"异常。为什么会这样？看源代码：

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

从上述程序中我们看到，当mLooper对象为空时，抛出该异常。这是因为该线程中的Looper对象还没创建，因此sThreadLocal.get()会返回null。Handler的原理就是要与MessageQueue建立关联，并且将消息投递给MessageQueue，如果连MessageQueue都没有，那么mHandler就没有存在的必要，而MessageQueue又被封装在Looper中，因此，创建Handler时Looper一定不能为空，解决办法如下：

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

Handler还有一个构造方法，是传入一个自定义的Looper：

```java
public Handler(Looper looper) {
    this(looper, null, false);
}
```

这样我们就可以在Handler中传入子线程的Looper，从而让子线程接收到消息，使用方法如下：

```java
Thread newThread = new Thread(new Runnable(){    
    @Override    
    public void run() {        
        Looper.prepare();        
        Looper.loop();    
    }
  });
newThread.start();
Handler handler = new Handler(newThread.getLooper());
```

但是上面的代码有一个问题，由于是子线程，什么时候执行run方法并不确定，也就是Looper.prepare()的调用时机也无法保证，所以，我们调用newThread.getLooper()时，可能为null。

**这个时候，HandlerThread就应运而生了**。它不仅可以帮我们创建相关的Looper，还可以解决这个异步导致的问题。接下来我们看主要代码：

```java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}

public Looper getLooper() {
    if (!isAlive()) {
        return null;
    }
    // If the thread has been started, wait until the looper has been created.
    synchronized (this) {
        while (isAlive() && mLooper == null) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
    }
    return mLooper;
}
```

上述代码可以看出，通过加锁，执行wait()和notifyAll()方法，解决了异步可能导致的问题。

HandlerThread的使用方法如下：

```java
HandlerThread thread = new HandlerThread("test");
thread.start();
Handler handler = new Handler(thread.getLooper()) {
    @Override
    public void handleMessage(Message msg) {
        System.out.println("current thread = " + Thread.currentThread());
    }
};
handler.sendEmptyMessage(1);
```

* 创建一个HandlerThread对象，并传入一个name参数。
* 创建Handler，传入HandlerThread的Looper对象。
* 调用handler的相关方法发送消息，之后就可以在子线程中处理相关消息了。

最后不使用了，记得调用quit()或者quitSafely()方法来终止线程的执行，其实就是终止loop的死循环，使得线程执行完全部代码：

```java
public boolean quit() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quit();
        return true;
    }
    return false;
}
public boolean quitSafely() {
    Looper looper = getLooper();
    if (looper != null) {
        looper.quitSafely();
        return true;
    }
    return false;
}
```

***

***

***

### IntentService

#### 特点

* IntentService是一种特殊的Service，它继承了Service，并且是一个抽象类，因此必须创建它的子类才能使用IntentService。
* IntentService可用于执行后台耗时任务，当任务执行结束后它会自动停止。
* 由于IntentService是服务的原因，它的优先级比线程高，所以适合执行一些高优先级的后台任务。
* IntentService封装了HandlerThread和Handler。
* 省去了Service创建线程的那些麻烦。
* 注意：IntentService的onHandlerIntent方法是在子线程中回调，更新UI需要去主线程处理。

#### 使用场景

　　假设我们碰到这样的需求，一项任务分为几个子任务，子任务需要按顺序先后执行，子任务全部执行完成后，任务才算成功。那么，利用几个子线程顺序执行是可以满足这个需求的，但是每个线程都需要手动去控制，而且得在一个子线程执行完成后才能开启另一个子线程。或者，将任务全部放到一个线程中，让其顺序执行，这样也可以满足需求。但是，如果这是一个后台任务，就得放在Service里面，由于Service运行在主线程，执行耗时任务需要在Service中开启子线程来执行。那么，有没有一种简单的方法处理呢，答案就是**IntentService**。

#### 使用方法

1. 创建一个类，继承IntentService，并重写onHandleIntent(intent)方法，该方法是IntentService的一个抽象方法，用来处理我们startService方法开启的服务，传入的intent就是我们startService是传入的intent。

   ```java
   public class MyIntentService extends IntentService {
       public MyIntentService() {
           super("[name]"); // name用于命名工作线程，仅对调试有意义
       }
       @Override
       protected void onHandleIntent(Intent intent) {
           int index = intent.getIntExtra("index", 0);
           System.out.println("这是第 " + index + " 个任务");
       }
   }
   ```

2. 清单文件注册该服务

   ```java
   <service android:name=".MyIntentService"/>
   ```

3. 启动该服务

   ```java
   Intent intent = new Intent(this, MyIntentService.class);
   for (int i = 0; i < 5; i++) {
       intent.putExtra("index", i);
       startService(intent);
   }
   ```

4. 运行结果如下：

   ```
   2020-04-29 22:03:51.673 29514-29601/com.nj.testappli I/System.out: 这是第 0 个任务
   2020-04-29 22:03:51.704 29514-29601/com.nj.testappli I/System.out: 这是第 1 个任务
   2020-04-29 22:03:51.707 29514-29601/com.nj.testappli I/System.out: 这是第 2 个任务
   2020-04-29 22:03:51.722 29514-29601/com.nj.testappli I/System.out: 这是第 3 个任务
   2020-04-29 22:03:51.722 29514-29601/com.nj.testappli I/System.out: 这是第 4 个任务
   ```

#### 源码分析

IntentService封装了HandlerThread和Handler，这一点从它的onCreate方法可以看出：

```java
public void onCreate() {
    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();
    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```

* 当IntentService被第一次启动的时候，onCreate方法才会被调用，onCreate方法中创建了一个HandlerThread，然后使用他的Looper构建了一个Handler对象`mServiceLooper`，这样通过`mServiceHandler`发送的消息，最终会在HandlerThread中执行，所以IntentService可用于执行后台任务。
* 每次启动IntentService，都会调用onStartCommand方法，该方法中处理每个后台任务的Intent，onStartCommand方法中又调用了onStart方法。

```java
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```

* IntentService仅仅是通过`mServiceHandler`发送了一个消息，该消息会在HandlerThread中被处理。这个过程中传递了Intent对象，也就是startService(intent)中的intent。通过该intent就可以解析出外界启动IntentService时所传递的参数，用来区分具体的后台任务。

```java
private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }
    @Override
    public void handleMessage(Message msg) {
        onHandleIntent((Intent)msg.obj);
        stopSelf(msg.arg1);
    }
}
```

* Handler中会调用onHandleIntent方法，并继续传递intent参数，该方法是一个抽象方法，需要我们在子类中去实现。
* onHandleIntent方法执行结束后，会执行stopSelf(int startId)方法来尝试停止服务。这里没有采用stopSelf()是因为stopSelf()会立即停止服务，而这个时候可能还有消息未处理，stopSelf(int startId)会等待所有的消息处理完毕后才终止服务。
* stopSelf(int startId)在尝试停止服务之前会判断最近启动的服务的次数是否和startId相等，如果相等就立即停止服务，不相等则不停止服务。

#### 总结

* IntentService的onHandleIntent方法是一个抽象方法，需要子类去实现，它的作用是根据Intent的参数区分具体的任务并执行。
* 如果只存在一个后台任务，那么onHandleIntent方法执行完这个任务后，调用stopSelf(int startId)直接停止服务。
* 如果有多后台任务， 当onHandleIntent执行完最后一个任务时，才会停止服务。
* 每执行一个后台任务必须启动一次IntentService。
* IntentService内部是通过消息的方式向HandlerThread请求执行任务，Handler中的Looper是顺序处理消息，以为这多个任务同时存在时，后台任务会按照外发起的顺序排队执行。

**一句话总结：比线程优先级高，不容易被杀死；使用子线程，不阻塞主线程，可执行耗时任务；以串行的方式执行任务，任务结束可自动销毁**。

***

***

***

### ThreadPoolExecutor

[线程池](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/JavaThread.md#四-线程池)

***

***

***

### 参考链接

* [为什么要用HandlerThread,怎么用?](https://www.jianshu.com/p/f0cdea1c232a)