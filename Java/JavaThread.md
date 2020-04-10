## Java线程

### 目录

* [一. 线程基础]()
  * [1. 进程与线程]()
  * [2. 线程的状态]()
  * [3. 线程的创建]()
  * [4. 线程的终止]()
* [二. 多线程]()
  * [1. 线程间的协作]() 
  * [2. 多线程相关方法——Callable、Future和FutureTask]()
* [三. 同步]()
  * [同步锁]() 
  * [同步集合]()
  * [阻塞队列]()
* [四. 线程池]()
  * []()
  * []()
  * []()
  * []()

***

***

***

### 一. 线程基础

#### 1. 进程与线程

* **进程**

  　　进程是操作系统结构的基础，是程序在一个数据集合上运行的过程，是系统进行资源分配和调度的基本单位。进程可以被看作程序的实体，它也是线程的容器。

* **线程**

  　　线程是操作系统调度的最小单元，也叫做轻量级进程。每个线程都拥有自己的计数器、堆栈和局部变量等属性，并且能够访问共享的内存变量。

* **为何使用多线程**

  * 使用多线程可以减少程序的响应时间。如果某个操作很耗时，或者陷入长时间的等待，此时程序将不会响应鼠标和键盘等的操作，使用多线程后可以把这个耗时操作分配到一个单独的线程中去执行，从而使程序具备了更好的交互性。
  * 与进程相比，线程创建和切换开销更小，同时多线程在数据共享方面效率非常高。
  * 多CPU或者多核计算机就具备执行多线程的能力。使用多线程可以提高CPU利用率。
  * 使用多线程能简化程序的机构，使程序便于理解和维护。

***

#### 2. 线程的状态

* **New**：新创建状态。线程被创建，还没有调用start方法，在线程运行之前还有一些基础工作要做。
* **Runnable**：可运行状态。一旦调用start方法，线程就处于Runnable状态。一个可运行的线程可能正在运行也可能没有运行，这取决于操作系统给线程提供运行的时间。
* **Blocked**：阻塞状态。表示线程被锁阻塞，它暂时不活动。
* **Waiting**：等待状态。线程暂时不活动，并且不运行任何代码，这消耗最少的资源，直到线程调度器再次激活它。
* **Timed waiting**：超时等待状态。和等待状态不同，它是可以在指定的时间自行返回的。
* **Terminated**：终止状态。表示当前线程已经执行完毕。导致线程终止有两种情况：第一种使run方法执行完毕正常退出；第二种是因为一个没有捕获的异常而终止了run方法，导致线程终止。

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/threadstatuschange.png" alt="图：线程状态切换" style="zoom:60%;" />

***

#### 3. 线程的创建

* **继承Thread类，重写run()方法**

  Thread本质是实现了Runnbale接口的一个实例。

  **注意**：调用start()方法后并不是立即执行多线程的代码，而是使该线程变为可运行状态，什么时候运行由操作系统决定。

  (1) 定义Thread类的子类，并重写run()方法。

  (2) 创建Thread子类的实例，即创建线程对象。

  (3) 调用线程对象的start()方法来启动该线程。

  ```java
  public class TestThread extends Thread {
      public void run() {
          System.out.println("Hello World");
      }
      public static void main(String[] args) {
          Thread thread = new TestThread();
          thread.start();
      }
  }
  ```

* **实现Runnable接口，并实现该接口的run()方法**

  (1) 自定义病史先Runnable接口，实现run()方法。

  (2) 创建Thread子类的实例，用实现Runnable接口的对象作为参数实例化该Thread对象。

  (3) 调用Thead的start()方法来启动该线程。

  ```java
  class TestRunnable implements Runnable {
      @Override
      public void run() {
          System.out.println("Hello World");
      }
  }
  public class TestRunnbale {
      public static void main(String[] args) {
          TestRunnable runnable = new TestRunnable();
          Thread thread1 = new Thread(runnable);
          thread1.start();
      }
  }
  ```

* **实现Callable接口，重写call()方法**

  Callable接口属于Executor框架的功能类，比Runnable功能更强大，主要表现为以下3点：

  (1) Callable可以在任务接受后提供一个返回值，Runnable不可以。

  (2) Callable中的call()方法可以抛出异常，而Runnable的run()方法不能抛出异常

  (3) 运行Callable可以拿到一个Feture对象，Feture对象表示异步计算的结果，它提供了检查计算是否完成的方法。由于线程属于异步计算模型，因此无法从别的线程中得到函数的返回值，在这种情况下就可以使用Feture来监视目标线程调用call()方法的情况。但调用Future的get()方法以获取结果时，当前线程就会阻塞，直到call()方法返回结果。

  ```java
  public class TestCallable {
      public static class MyTestCallable implements Callable {
          public String call() throws InterruptedException {
              return "Hello World";
          }
      }
      public static void main(String[] args) {
          MyTestCallable myTestCallable = new MyTestCallable();
          ExecutorService executorService = Executors.newSingleThreadExecutor();
          // submit方法启动该线程
          Future future = executorService.submit(myTestCallable);
          try {
              // 获取请求结果
              System.out.println(future.get());
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  }
  ```

***

#### 4. 线程的终止

* 中断

  * `Thread.currentThread.interrupt()`将线程的中断标识位置为true。线程会时不时的检查中断标识位。

  * `Thread.currentThread.isInterrupted()`查看是否被置位。

  * `Thread.interrupted()`对中断标识位进行复位。

  * 如果一个线程被阻塞，就无法检测中断状态。

  * 如果一个线程处于阻塞状态，线程在检查中断标识位时如果发现中断标识位为true，则会在阻塞方法出抛出InterruptedException异常，并且抛出异常前将中断标识位复位，设置为false。被中断的现成不一定会终止，中断是为了引起线程的注意，可以捕获异常进行处理。

    ```java
    public class TestCallable extends Thread {
        @Override
        public void run() {
            super.run();
            try {
                sleep(2000); // 外部设置了中断标识位，这里会抛出异常
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        public static void main(String[] args) throws InterruptedException {
            Thread thread1 = new TestCallable();
            thread1.start();
            Thread.sleep(1000);
            thread1.interrupt();
        }
    }
    ```

    **注**：中断可以用来解决"线程由于阻塞或者长时间执行不能结束"的问题。

* 安全的终止线程

  * 采用boolean变量来控制是否需要停止线程

    ```java
    public class StopThread {
        public static void main(String[] args) throws InterruptedException {
            MoonRunnable runnable = new MoonRunnable();
            Thread thread = new Thread(runnable, "MoonThread");
            thread.start();
            TimeUnit.MILLISECONDS.sleep(10);
            runnable.cancel();
        }
        public static class MoonRunnable implements Runnable {
            private long i;
            private volatile boolean on = true;
            @Override
            public void run() {
                while (on) {
                    i++;
                }
            }
            public void cancel() {
                on = false;
            }
        }
    }
    
    ```

  * 采用检查中断来终止线程

    ```java
    public class StopThread {
        public static void main(String[] args) throws InterruptedException {
            MoonRunnable runnable = new MoonRunnable();
            Thread thread = new Thread(runnable, "MoonThread");
            thread.start();
            TimeUnit.MILLISECONDS.sleep(10);
            thread.interrupt();
        }
        public static class MoonRunnable implements Runnable {
            private long i;
            @Override
            public void run() {
                while (Thread.currentThread().isInterrupted()) {
                    i++;
                }
            }
        }
    }
    ```

***

***

***

### 二. 多线程

#### 1. 线程间的协作

| 函数  | 作用                                                         |
| ----- | ------------------------------------------------------------ |
| wait  | 当一个线程执行到wait方法时，它就进入到一个和该对象相关的等待池中，同时失去（释放）了对象的机锁（**线程没有锁，对象才有锁**），使得其他线程可以访问。用户可以使用notify、notifyAll或者指定睡眠时间来唤醒当前等待池中的线程。<br />**注意**：wait、notify、notifyAll必须放在synchronized block中，否则会抛出异常。 |
| sleep | 该函数是Thread的静态函数，作用是使调用线程进入睡眠状态。因为sleep是Thread类的static方法，因此他不能改变对象的机锁。所以，当在一个synchronized块中调用sleep方法时，线程虽然休眠了，但是对象的机锁并没被释放，其他线程无法访问这个对象。 |
| join  | 等待目标线程执行完成之后再继续执行                           |
| yield | 线程礼让。目标线程由运行状态转换为就绪状态，也就是让出执行权限，让其他线程得以优先执行，但其他线程能否优先执行是未知的。 |

* wait和notify、notifyAll的运用

  先说两个概念，Java中每个对象都有两个池，**锁池**和**等待池**

  > **锁池**：假设线程A已经拥有了某个对象(注意:不是类)的锁，而其它的线程想要调用这个对象的某个synchronized方法(或者synchronized块)，由于这些线程在进入对象的synchronized方法之前必须先获得该对象的锁的拥有权，但是该对象的锁目前正被线程A拥有，所以这些线程就进入了该对象的锁池中。
  >
  > **等待池**：假设一个线程A调用了某个对象的wait()方法，线程A就会释放该对象的锁后，进入到了该对象的等待池中。





* join

  官方解释：阻塞当前调用join函数时所在的线程，直到接收线程执行完毕之后再继续。

  ```java
  public static void main(String[] args){
      Worker worker=new Worker();
      worker.start();
      try{
          // 主线程会阻塞，直到worker执行完成
          worker.join();
      }catch(InterruptedException e){
          e.printStackTrace();
      }
  }
  static class Worker extends Thread {
      @Override
      public void run() {
          try {
              Thread.sleep(2000);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  }
  ```

* yield

  官方解释：使调用该函数的线程主动让出执行时间给其他已就绪状态的线程。

  ```java
  public class TestJoin {
      public static void main(String[] a) {
          YieldThread thread1 = new YieldThread("thread-1");
          YieldThread thread2 = new YieldThread("thread-2");
          thread1.start();
          thread2.start();
      }
      static class YieldThread extends Thread {
          public YieldThread(String name) {
              super(name);
          }
          public void run() {
              for (int i = 0; i < 5; i++) {
                  System.out.println(getName() + " -> " + i);
                  if (i == 2) {
                      Thread.yield();
                  }
              }
          }
      }
  }
  ```

  理论上的输出结果是：

  ```
  thread-1 -> 0
  thread-1 -> 1
  thread-1 -> 2
  thread-2 -> 0
  thread-2 -> 1
  thread-2 -> 2
  thread-1 -> 3
  thread-1 -> 4
  thread-2 -> 3
  thread-2 -> 4
  ```

  但实际并不是上面的结果，结果也是很多种可能性。因为thread-1线程调用yield让出cpu之后，进入了runnable状态，也就是说该线程和其他线程一起竞争cpu，所以，cpu很有可能再次把时间片分配给thread-1线程。而且每次时间片的分配时间也和分配对象都不固定。（多核的处理器，只创建两个线程，是看不出效果的，可以尽量多的创建线程，就可以看出大概的分配趋势）

***

#### 2. 多线程相关方法——Callable、Future和FutureTask

Callable、Future、futureTask只能运用在线程池，而Runnable即能运用在Thread中，也能运用在线程池中。

* **Callable**有返回值，Runnable没有返回值。

  ```java
  public interface Callable<V> {
      V call() throws Exception;
  }
  ```

* **Future为线程池制定了一个可管理的任务标准。它提供了对Runnable或者Callable的管理操作。**

  ```java
  public interface Future<V> {
      // 取消任务
      boolean cancel(boolean mayInterruptIfRunning);
      // 任务是否已经被取消
      boolean isCancelled();
      // 任务是否已经完成
      boolean isDone();
      // 获取结果，如果任务未完成，则等待，直到完成。该函数会阻塞
      V get() throws InterruptedException, ExecutionException;
      // 获取结果，如果还未完成则等待，直到timeout或者返回结果。该函数会阻塞
      V get(long timeout, TimeUnit unit)
              throws InterruptedException, ExecutionException, TimeoutException;
  }
  ```

* **FutureTask**是Future的实现类

  ```java
  public class FutureTask<V> implements RunnableFuture<V> {
      public FutureTask(Callable<V> var1) {
          if (var1 == null) {
              throw new NullPointerException();
          } else {
              this.callable = var1;
              this.state = 0;
          }
      }
      public FutureTask(Runnable var1, V var2) {
          this.callable = Executors.callable(var1, var2);
          this.state = 0;
      }  
  }  
  
  public interface RunnableFuture<V> extends Runnable, Future<V> {
      void run();
  }
  ```

  上述代码可以看出，FurureTask同时具备了Runnable和Future两个接口的能力。FutureTask会像Thread包装Runnable那样对Runnable和Callable进行包装，由构造函数注入。

  上述代码可以看出，注入的Runnable会被转换为Callable，所以FutureTask最终执行的是Callable的任务。

  ```java
  public static <T> Callable<T> callable(Runnable var0, T var1) {
      if (var0 == null) 
          throw new NullPointerException();
      return new Executors.RunnableAdapter(var0, var1);
  }
  // Runnable适配器，将Runnable转换为Callable
  static final class RunnableAdapter<T> implements Callable<T> {
      final Runnable task;
      final T result;
      RunnableAdapter(Runnable var1, T var2) {
          this.task = var1;
          this.result = var2;
      }
      public T call() {
          this.task.run();
          return this.result;
      }
  }
  ```

* [Runnable、Callable、FutureTask使用案例]()

***

***

***

### 三. 同步

#### 1. 同步锁

* **synchronized**

  ```java
  public class SynchronizedTest {
      public synchronized void syncMethod() { // 同步方法
      }
      public void syncThis() { // 同步块
          synchronized (this) { }
      }
      public void syncClassMethod() { // 同步class对象
          synchronized (SynchronizedTest.class) { }
      }
      public synchronized static void syncStaticMethod() { // 同步静态方法
      }
  }
  ```

  `syncMethod`和`syncThis`锁的是对象，`syncClassMethod`和`syncStiticMethod`锁的是class对象。

  * 作用与引用对象，是防止多个线程同时访问添加了synchronized锁的代码块。
  * 作用于class对象，是防止其他线程访问同一个对象中的synchronized代码块或者函数。

  **Java中每一个对象都有一个内部锁，并且该锁有一个内部条件**。由该锁来管理那些试图进入synchronized方法的线程，由该锁的条件来管理那些调用wait的线程。

* **ReentrantLock**

  * 可重入锁，表示该锁支持一个线程对资源的重复加锁。ReentrantLock的基本操作表如下：

    | 函数                             | 作用                                         |
    | -------------------------------- | -------------------------------------------- |
    | lock()                           | 获取锁                                       |
    | tryLock()                        | 尝试获取锁                                   |
    | tryLock(long time,TimeUnit unit) | 尝试获取锁，如果指定时间还获取不到，那么超时 |
    | unlock()                         | 释放锁                                       |
    | newCondition()                   | 获取锁的Condition                            |

  * ReentrantLock的常用形式如下：

    ```java
    Lock mLock = new ReentrantLock();
    mLock.lock();
    try{
    ...
    }
    finally{
    mLock.unlock();
    }
    ```

    **注**：把解锁的操作放在finally中，是保证临界区发生异常后，能够顺利的释放锁，避免阻塞，造成死锁。

  * Condition

    在ReentrantLock中有一个重要的函数`newCondition()`，用于获取一个条件对象。Condition用于实现线程间通信。Condition方法表如下：

    | 函数                          | 作用                           | synchronized对应方法 |
    | ----------------------------- | ------------------------------ | -------------------- |
    | await()                       | 线程等待                       | wait()               |
    | await(int time,TimeUnit unit) | 线程等待特定时间，超过则为超时 |                      |
    | singnal()                     | 随机唤醒某个等待线程           | notify()             |
    | singnalAll()                  | 唤醒所有等待中的线程           | notifyAll()          |

    ```java
    public void transfer(int from, int to, double amount) throws InterruptedException {
        alipaylock.lock();
        try {
            while (accounts[from] < amount) {
                // 阻塞当前线程，释放锁
                condition.await();
            }
            // 转账操作
            accounts[from] = accounts[from] - amount;
            accounts[to] = accounts[to] + amount;
            condition.signalAll();
        }finally {
            alipaylock.unlock();
        }
    }
    ```

    上面是Condition的使用方法，相对来说synchronized关键字来编写代码，就简单多了，只需要给transfer方法加上synchronized关键字、condition.await()换成wait()、condition.signalAll()换成notifyAll()就可以了。

  * [案例：ReentrantLock和Condition实现简单阻塞队列MyArrayBlockingQueue]()

* **synchronized和ReentrantLock的比较**

  * 使用同步，JVM将确保锁会自动释放。Lock类需要手动释放
  * 使用同步，JVM在生成线程转储时能够包括锁定信息，这些对调试非常有用，因为它们能标识死锁或者其他异常行为的来源。JVM不知道具体哪个线程拥有Lock对象。

  |        | synchronized | ReentrantLock |
  | ------ | ------------ | ------------- |
  | 锁释放 | 自动释放     | 手动释放      |
  |        |              |               |
  |        |              |               |
  |        |              |               |

* **信号量Semaphore**

  　　Semaphore是一个计数信号量，它的本质是一个"共享锁"。信号量维护一个信号量许可集，调用`acquire()`来获取信号量许可。当信号量中有可用许可时，线程获取该许可，否则线程必须等待，直到有可用的许可。线程通过`release()`来释放它所持有的许可。

  　　假设银行有3个窗口，要办业务5个人，同时只能有3个人办业务，另外两个人等待。许可集为3，占用窗口的人调用`acquire()`获取许可，离开时调用`release()`释放许可，后续的人才能获取许可。

  * [Semaphore的示例]()

* **循环栅栏CyclicBarrier**

  　　CyclicBarrier是一个同步辅助类，允许一组线程互相等待，直到到达某个公共屏障点。

  * [CyclicBarrier的示例]()

* **闭锁CountDownLatch**

  　　CountDownLatch也是一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待，直到条件被满足。

  * [CountDownLatch的示例]()
  * CountDownLatch和CyclicBarrier的不同点：
    *  CountDownLatch的作用是允许1个或者N个线程等待其他线程完成执行，而CyclicBarrier则是允许N个线程互相等待。
    *  CountDownLatch的计数器无法被重置，CyclicBarrier的计数器可以被重置后使用，因此它被称为是循环的barrier。

***

#### 2. 同步集合

* **CopyOnWriteArrayList**

  **思路**：从多个线程共享同一个列表，当某个线程想要修改这个列表的元素时，会把列表的元素Copy一份，然后进行修改，修改完成之后再将新的元素设置给这个列表，这是一种延时懒惰策略。

  **好处**：可以对CopyOnWrite容器并发的读，而不需要加锁，因为当前容器不会添加、移除任何元素。

  核心源码add方法的实现：

  ```java
  public boolean add(E e) {
      ReentrantLock lock = this.lock;
      lock.lock();
      try {
          Object[] elements = getArray();
          int len = elements.length;
          Object[] newElements = Arrays.copyOf(elements, len + 1);
          newElements[len] = e;
          setArray(newElements);
          return true;
      } finally {
          lock.unlock();
      }
  }
  ```

  添加元素时加锁，防止多线程写时Copy出N个副本出来。

  读的时候不需要加锁，如果读的时候有多个线程正在向CopyOnWriteArrayList添加数据，读还是会读到旧数据，因为写的时候不会锁住旧的元素数组，代码如下：

  ```java
  public E get(int index) {
      return get(getArray(), index);
  }
  ```

  **缺点**：在添加、移除元素时占用的内存空间翻了一倍，以时间换空间。相似的还有CopyOnWriteArraySet。

* **ConcurrentHashMap**

  HashTable是HashMap的线程安全实现。

  HashTable使用synchronized，多线程情况下效率低下。如线程1使用put方法，线程2不但不能使用put方法，并且不能使用get方法，所以效率低下。

  **所有访问HashTable的线程都必须竞争同一把锁**。

  假如容器中有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不存在锁竞争了，这就是**ConcurrentHashMap的锁分段技术**。

  如size()和containsValue()，它们可能需要锁定整个表而不仅是某个段，这时候需要按顺序锁定所有段，操作完毕后，又按顺序释放所有段的锁。

***

#### 3. 阻塞队列

* **概念**：阻塞队列常用于生产者和消费者的场景，生产者是往队列里添加元素的线程，消费者是从队列里拿元素的线程。阻塞队列就是生产者的容器，而消费者只从容器中拿元素。

* **阻塞场景**：

  (1) 当队列中没有数据的情况下，消费者端的所有线程都会被自动阻塞（挂起），直到有数据放入队列。

  (2) 当队列中填满数据的情况下，生产者端的所有线程都会被自动阻塞（挂起），直到队列有空的位置，线程被自动唤醒。

* **BlockingQueue**

  相关的方法：

  | 函数名             | 作用                                                         |
  | ------------------ | ------------------------------------------------------------ |
  | add(e)             | 把元素e加到BlockingQueue里，如果可以容纳，则返回true，否则抛出异常 |
  | offer(e)           | 将元素e加到BlockingQueue里，如果可以容纳，在返回true，否则返回false |
  | offer(e,time,unit) | 将元素e加到BlockingQueue里，如果可以容纳，则返回true，否则在等待指定的时间之后继续尝试添加，如果失败则返回false |
  | put(e)             | 将元素e加到BlockingQueue里，如果不能容纳，则调用此方法的线程被阻塞直到BlockingQueue里面有空间再继续添加 |
  | take()             | 取走BlockingQueue里排在队首的对象，若BlockingQueue为空，则进入等待状态直到BlockingQueue有新的对象被加入为止 |
  | poll(time,unit)    | 去除并移除队列中的队首元素，如果设定的阻塞时间内还没有获得数据，返回null |
  | element()          | 获取队首元素，如果队列为空，则抛出NoSuchElementException异常 |
  | peek()             | 获取队首元素，如果队列为空，返回null                         |
  | remove())          | 获取并移除队首元素，如果队列为空，则抛出NoSuchElementException异常 |

* **阻塞队列**

  Java中提供了7个阻塞队列，都实现了BlockingQueue接口，分别如下：

  * **ArrayBlockingQueue**：由数组结构组成的有界阻塞队列
  * **LinkedBlockingQueue**：由链表结构组成的有界阻塞队列
  * **PriorityBlockingQueue**：支持优先级排序的无界阻塞队列
  * **DelayQueue**：使用优先级队列实现的无界阻塞队列
  * **SynchronousQueue**：不存储元素的阻塞队列
  * **LinkedTransferQueue**：由链表结构组成的无界阻塞队列
  * **LinkedBlockingDeque**：由链接结构组成的双向阻塞队列

  [7个阻塞队列的详细介绍]()

  有界队列和无界队列的区别：

  * 有界队列：有固定大小的队列
  * 无界队列：无固定大小的队列。这些队列的特点是可以直接入列，直到溢出。现实中几乎不会有达到那么大容量的（Integer.MAX_VALUE），所以相当于无界。

***

***

***

### 四. 线程池

**优点**：

* (1) 重用存在的线程，减少对象创建、销毁的开销

* (2) 可有效控制最大并发数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞

* (3) 提供定时执行、定期执行、单线程、并发数控制等功能。

线程池都实现了ExecutorService接口。我们不会直接通过new的形式创建线程池，系统提供了Executors工厂类来简化这个过程。

ExecutorService的生命周期包括3种状态：

* 运行：创建后进入运行状态
* 关闭：调用shutdown()方法时，进入关闭状态，此时不接收新任务，但执行已提交的任务
* 终止：所以已提交的任务执行完成后，变成终止状态。

***

 #### 1. ThreadPoolExecutor

**功能**：启动指定数量的线程以及任务添加到一个队列中，并且将任务分发给空闲的线程。

ThreadPoolExecutor拥有最多参数的构造函数如下：

```java
public ThreadPoolExecutor(int corePoolSize, 
                          int maximumPoolSize,
                          long keepAliveTime, 
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory, 
                          RejectedExecutionHandler handler) 
```

* **corePoolSize**：核心线程数。

  * 线程池启动后默认为空，有任务来时才会创建线程。
  * prestartAllcoreThread方法可以在线程池启动时创建所有核心线程来等待任务。

* **maximumPoolSize**：线程池允许创建的最大线程数。

  * 如果任务队列满了并且线程数小于maximumPoolSize，则线程池会创建新的线程来处理任务。
  * 当workQueue使用无界队列（如LinkedBlockingQueue）时，此参数无效。
  * 如果corePoolSize和maximumPoolSize相同，则创建固定大小的线程池。

* **keepAliveTime**：非核心线程闲置的超时时间。

  * 非核心线程闲置超过这个时间则回收。

  * 如果任务多，每个任务执行时间很短，可以调大keepAliveTime来提高利用率。
  * 设置allowCoreThreadTimeOut的属性为true，keepAliveTime也可以应用到核心线程上。

* **TimeUnit**：keepAliveTime参数的时间单位。

* **workQueue**：任务队列。

  如果当前线程数大于corePoolSize，则将任务添加到任务队列中。该任务队列是BlockingQueue类型的，也就是阻塞队列。

* **ThreadFactory**：线程工厂。

  可以定制线程的创建过程，比如给线程设置名字。通常不需要设置此参数。

* **RejectedExecutionHandler**：饱和策略。

  * AbortPolicy：拒绝任务，并抛出RejectedExecutionException异常。**默认策略**。
  * CallerRunsPolicy：用调用者所在线程来处理任务。此策略提供简单的反馈控制机制，能减缓新任务的提交速度。
  * DiscardPolicy：添加不进去的任务都抛弃，没有异常抛出。
  * DiscardOldestPolicy：丢弃队列头部任务，并执行当前任务。

#### 2. 线程池处理流程和原理

```flow
op=>operation: 提交任务
op1=>operation: 创建核心线程执行任务
op2=>operation: 将任务加在任务队列中
op3=>operation: 创建非核心线程执行任务
op4=>operation: 饱和策略

cond=>condition: 线程数是否达到
								corePoolSize
cond1=>condition: 任务队列是否已满
cond2=>condition: 线程数是否达到
									最大线程数

op->cond
cond(yes)->cond1
cond(no)->op1
cond1(yes)->cond2
cond1(no)->op2
cond2(yes)->op4
cond2(no)->op3
```



***

***

***

### 附：案例代码

#### 1. Runnable、Callable、FutureTask使用案例

```java
public class FutureTest {
    static ExecutorService mExecutor = Executors.newSingleThreadExecutor();
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        futureWithRunnable();
        futureWithCallable();
        futureTask();
    }
    private static void futureWithRunnable() throws ExecutionException, InterruptedException {
        Future<?> result = mExecutor.submit(new Runnable() {
            @Override
            public void run() {
                fibc(20);
            }
        });
        System.out.println("runnable : " + result.get());
    }
    private static void futureWithCallable() throws ExecutionException, InterruptedException {
        Future<Integer> result2 = mExecutor.submit(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                return fibc(20);
            }
        });
        System.out.println("Callable : " + result2.get());
    }
    private static void futureTask() throws ExecutionException, InterruptedException {
        FutureTask<Integer> futureTask = new FutureTask(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                return fibc(20);
            }
        });
        mExecutor.submit(futureTask);
        System.out.println("FutureTask1 : " + futureTask.get());
    }
    private static void futureTaskR() {
        // FutureTask包装Runnable使用案例
    }
    private static int fibc(int num) {
        if (num == 0) {
            return 0;
        }
        if (num == 1) {
            return 1;
        }
        return fibc(num - 1) + fibc(num - 2);
    }
}
```

#### 2. ReentrantLock和Condition实现简单阻塞队列MyArrayBlockingQueue

```java
public class MyArrayBlockingQueue<T> {
    // 数据数组
    private final T[] items;
    // 锁
    private Lock lock = new ReentrantLock();
    // 队满的条件
    private Condition notFull = lock.newCondition();
    // 队空条件
    private Condition notEmpty = lock.newCondition();
    // 数据个数
    private int count;
    // 头部索引
    private int head;
    // 尾部索引
    private int tail;

    public MyArrayBlockingQueue(int size) {
        items = (T[]) new Object[size];
    }

    public MyArrayBlockingQueue() {
        this(10);
    }

    public void put(T t) {
        lock.lock();
        try {
            while (count == getCapacity()) {
                // 已经是满的，阻塞等待“不满”的线程
                System.out.println("数据已满，等待");
                notFull.await();
            }
            items[tail] = t;
            if (++tail == getCapacity()) {
                tail = 0;
            }
            ++count;
            // 已经确定不空了，唤醒 阻塞着的 等待“不空”的线程
            notEmpty.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public T take() {
        lock.lock();
        try {
            while (count == 0) {
                //已经是空的，阻塞等待“不空”的线程
                System.out.println("还没有数据，请等待");
                notEmpty.await();
            }
            T ret = items[head];
            items[head] = null;
            if (++head == getCapacity()) {
                head = 0;
            }
            --count;
            //已经确定不满了，唤醒 阻塞着的 等待“不满”的线程
            notFull.signalAll();
            return ret;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
        return null;
    }

    public int getCapacity() {
        return items.length;
    }
}

    public static void main(String[] args) {
        MyArrayBlockingQueue<Integer> aQueue = new MyArrayBlockingQueue<>();
        aQueue.put(3);
        aQueue.put(24);
        for (int i = 0; i < 5; i++) {
            System.out.println(aQueue.take());
        }
    }
```

#### 3. Semaphore的示例

```java
public class SemaphoreTest {
    public static void main(String[] args) {
        final ExecutorService service = Executors.newFixedThreadPool(3);
        final Semaphore semaphore = new Semaphore(3);
        for (int i = 0; i < 5; i++) {
            service.submit(new Runnable() {
                @Override
                public void run() {
                    try {
                        semaphore.acquire();
                        System.out.println("剩余许可 ： " + semaphore.availablePermits());
                        Thread.sleep(2000);
                        semaphore.release();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }
}

剩余许可 ： 2
剩余许可 ： 1
剩余许可 ： 0
剩余许可 ： 2 // 2s后输出
剩余许可 ： 1
```

#### 4. CyclicBarrier的示例

```java
public class CyclicBarrierTest {
    private static final int SIZE = 5;
    private static CyclicBarrier mCyclicBarrier;
    public static void main(String[] args) {
        mCyclicBarrier = new CyclicBarrier(SIZE, new Runnable() {
            @Override
            public void run() {
                System.out.println(" --> 满足条件，参与者：" + mCyclicBarrier.getParties());
            }
        });
        for (int i = 0; i < SIZE; i++) {
            new WorkerThread().start();
        }
    }
    static class WorkerThread extends Thread {
        public void run() {
            try {
                System.out.println(Thread.currentThread().getName() + " 等待 CyclicBarrier");
                mCyclicBarrier.await();
                System.out.println(Thread.currentThread().getName() + " 继续执行");
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}

Thread-0 等待 CyclicBarrier
Thread-1 等待 CyclicBarrier
Thread-2 等待 CyclicBarrier
Thread-3 等待 CyclicBarrier
Thread-4 等待 CyclicBarrier
 --> 满足条件，参与者：5
Thread-4 继续执行
Thread-2 继续执行
Thread-1 继续执行
Thread-0 继续执行
Thread-3 继续执行
```

只有当5个线程都调用了mCyclicBarrier.await()函数之后，后续的代码才会执行。

CyclicBarrier可以用于多个线程等待，知道某个条件被满足。

#### 5. CountDownLatch的示例

```java
public class CountDownLatchTest {
    private static int LATCH_SIZE = 5;
    public static void main(String[] args) {
        try {
            CountDownLatch latch = new CountDownLatch(LATCH_SIZE);
            for (int i = 0; i < LATCH_SIZE; i++) {
                new WorkerThread(latch).start();
            }
            System.out.println("主线程等待");
            // 主线程等待池中5个任务的完成
            latch.await();
            System.out.println("主线程继续执行");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    static class WorkerThread extends Thread {
        CountDownLatch mLatch;
        public WorkerThread(CountDownLatch latch) {
            mLatch = latch;
        }
        public void run() {
            try {
                Thread.sleep(1000);
                System.out.println(Thread.currentThread().getName() + " 执行操作");
                // 将CountDownLatch的数值减1
                mLatch.countDown();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}

主线程等待
Thread-2 执行操作 // 1s后执行
Thread-0 执行操作
Thread-1 执行操作
Thread-4 执行操作
Thread-3 执行操作
主线程继续执行
```

#### 6. 7个阻塞队列的详细介绍

* **ArrayBlockingQueue**

  * 由数组结构组成的有界阻塞队列

  * 按照FIFO原则队元素进行排序

  * 默认情况下不保证线程公平地访问队列，通常为了保证公平性会降低吞吐量

  * 代码创建一个公平的阻塞队列

    ```java
    ArrayBlockingQueue queue = new ArrayBlockingQueue(2000, true);
    ```

* **LinkedBlockingQueue**

  * 由链表结构组成的有界阻塞队列
  * 按照FIFO原则队元素进行排序
  * 对生产者和消费者分别采用了独立的锁来控制数据同步，所以高效。
  * 构造LinkedBlockingQueue对象时，如果未指定其容量大小，会默认一个类似于无限大小的容量。

* **PriorityBlockingQueue**

  * 支持优先级排序的无界阻塞队列
  * 默认情况下元素才去自然顺序升序排列。
  * 自定义实现compareTo()方法来指定元素进行排序规则
  * 初始化PriorityBlockingQueue时，指定构造参数Comparator来队元素进行排序。
  * 不能保证同优先级元素的顺序。

* **DelayQueue**

  * 使用优先级队列实现的无界阻塞队列
  * 队列使用PriorityQueue来实现
  * 队列中的元素必须实现Delayed接口。
  * 创建元素时，可以指定元素到期的时间，只有在元素到期时才能从队列中取走。

* **SynchronousQueue**

  * 不存储元素的阻塞队列
  * 每个插入操作必须等待另一个线程的移除操作，每个移除操作必须等待另一个线程的插入操作。因此队列中没有元素，容量为0。
  * 队列没有容量，不能调用peek操作。

* **LinkedTransferQueue**

  * 由链表结构组成的无界阻塞队列
  * 实现接口TransferQueue，该接口有5个方法。

* **LinkedBlockingDeque**

  * 由链接结构组成的双向阻塞队列
  * 从队列两端插入和移除元素，减少一半竞争。
  * 多了addFirst、addLast、offerFirst、offerLast、peekFirst、peekLast等方法。