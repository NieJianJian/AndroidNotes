## ThreadLocal详解

### 1. ThreadLocal是什么

　　ThreadLocal可以理解为是用来存储线程局部变量，只有当前线程才能访问的数据，从而也就不会存在线程不安全的问题了。

### 2. ThreadLocal的使用

```java
ThreadLocal<T> threadLocal = new ThreadLocal<T>();
T t = threadLocal.get();
threadLocal.set(new T());
threadLocal.remove();
```

通过get和set方法来使用ThreadLocal。

* 一个ThreadLocal对象对应存储一个值，因为底层是用Key-value的方式存储，key是ThreadLocal对象，value是set传递的值。
* 如果需要储存多个值，就需要创建多个ThreadLocal对象，如果直接多次调用同一个ThreadLocal的set方法，存储的值将会不断的更新，为最后一次set的值。
* 如果线程不消亡，这些本地变量就会一直存在，所以不再使用的时候，可以将其remove掉。

### 3. ThreadLocal源码分析

首先来看几个重要的方法：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

从上面的代码，可以得出几个结论：

* get和set方法，最终操作的都是ThreadLocalMap类型的变量`map`。所以可以最终的值的存取不是在ThreadLocal，而是在ThreadLocalMap中。
* getMap获取的是当前线程的`threadLocals`变量。说明，一个线程只有一个ThreadLocalMap对象。
* 首次get方法时，如果之前没有任何set操作，将初始化一个默认值，value为null，存储到`map`中。
* 不管是首次get还是set操作，最终都会调用createMap方法，初始化`map`对象。并赋值给当前线程的`threadLocals`变量。

接下来看ThreadLocalMap的主要内容:

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
    private static final int INITIAL_CAPACITY = 16;
    private Entry[] table;
    private int size = 0;
    private int threshold; // Default to 0
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
    
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
    
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }

    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
        while (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == key)
                return e;
            if (k == null)
                expungeStaleEntry(i);
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }

    private void set(ThreadLocal<?> key, Object value) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len - 1);
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();
            if (k == key) {
                e.value = value;
                return;
            }
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
}
```

有上述代码，可以看出：

* ThreadLocalMap是通过Entry进行存储的，实例化的时候创建了一个长度为16的Entry数组。

* Entry采用的弱引用。

* 使用当前的ThreadLocal对象作为key进行存储。所以，想要存储多个变量，需要多次创建ThreadLocal，以用来存储不同的值。

* set方法中可以看到，value要存储在Entry数组中的位置，是通过运算得到的index。

  * 首先通过hash运算，得到index

  * 根据index得到对应的Entry，判断该index处是的Entry否为null；

  * 如果不为null，则判断这次需要存储的key和Entry的key是否一致，如果一致则更新数据，return；

  * 如果Entry不为null，获取到的k却为null。则对数据进行一次清理，然后将这次的key-value插入。

  * 如果获取到的index对应的Entry为null，则直接在当前位置插入新数据。

  * 插入数据后判断当前的数据大小，是否超过阀值（总容量的2/3），如果超过，则调用rehash

  * reHash方法如下：

    ```java
    private void rehash() {
        expungeStaleEntries();
        // Use lower threshold for doubling to avoid hysteresis
        if (size >= threshold - threshold / 4)
             resize();
        }
    }
    ```

    首先是要再做一次清理，清除陈旧数据。如果清除之后，数据大小还是大于阈值的3/4，则调用reSize。

  * reSize方法将容量扩大为原来的2倍。

* 和HashMap的最大的不同在于，ThreadLocalMap结构非常简单，没有next引用，也就是说ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用**线性探测**的方式，所谓线性探测，就是根据初始key的hashcode值确定元素在table数组中的位置，如果发现这个位置上已经有其他key值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。

  ThreadLocalMap解决Hash冲突的方式就是简单的步长加1或减1，寻找下一个相邻的位置。

  ```java
  private static int nextIndex(int i, int len) {
      return ((i + 1 < len) ? i + 1 : 0);
  }
  
  private static int prevIndex(int i, int len) {
      return ((i - 1 >= 0) ? i - 1 : len - 1);
  }
  ```

  显然ThreadLocalMap采用线性探测的方式解决Hash冲突的效率很低，如果有大量不同的ThreadLocal对象放入map中时发送冲突，或者发生二次冲突，则效率很低。

* 由于ThreadLocalMap的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。这也是为什么需要需要多次清理陈旧数据。

* ThreadLocalMap的get、set、remove方法中，都实现了对key为null的Entry的清除。但仍可能会发生内存泄露，因为可能使用了ThreadLocal的get或set方法后发生GC，此后不调用get、set或remove方法，为null的value就不会被清除。

  解决办法是每次使用完ThreadLocal都调用它的remove()方法清除数据，或者按照JDK建议将ThreadLocal变量定义成private static，这样就一直存在ThreadLocal的强引用，也就能保证任何时候都能通过ThreadLocal的弱引用访问到Entry的value值，进而清除掉。

### 4. InheritableThreadLocal详解

同一个ThreadLocal对象，在不同的线程中访问的值是不一样的，即使是子线程访问父线程，数据也获取不到。

如果我们希望子线程可以访问父线程的数据，可以使用InheritableThreadLocal。

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    protected T childValue(T parentValue) {
        return parentValue;
    }
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

* InheritableThreadLocal继承自ThreadLocal
* 重写了childValue、getMap和createMap方法
* getMap中返回的不是`threadLocals`，而是`inheritableThreadLocals`；
* createMap创建的ThreadLocalMap也是赋值给了`inheritableThreadLocals`变量。

用户创建Thread

```java
Thread thread = new Thread();
```

Thread的创建

```java
public Thread() {
    this.init((ThreadGroup)null, (Runnable)null, "Thread-" + nextThreadNum(), 0L);
}
```

Thread的初始化

```java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
    ......（其他代码）
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    ......（其他代码）
}
```

最终调用ThreadLocal的createInheritedMap方法进行初始化。

```java
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

在构造方法中，将父线程的parentMap的值，逐一复制到自身线程中。

*******

***

***

**严格的说，Java中不存在实质上的父子关系**

没有方法可以获取一个线程的父线程，也没有方法可以获取一个线程所有的子线程

子线程的消亡与父线程的消亡并没有任何关系，不会因为父线程的结束而导致子线程退出（操作系统中如此）。

可以看得出来，在init方法中，将创建这个线程的当前线程定义为“父”

```java
 Thread parent = currentThread();
```

在初始化之后，线程组（如果没设置）、是否为守护线程、优先级、上下文类加载器、父线程ThreadLocal（稍后讲解）都是从当前线程获取的

除了一些初始值的设置来自于所谓“父线程”之外，并没有强关系

**所以说，对Java中的线程，父线程的概念，只是一种逻辑称呼，创建线程的当前线程就是新线程的父线程，新线程的一些资源来自于这个父线程**

**在init方法中，对于所谓父线程的处理逻辑，换一个说法就是借助于当前正在运行的线程，对新创建线程进行一些必要的赋值与初始化**

结论：**父线程的生命周期与子线程没有关系。**

个人感觉：每个线程包括main线程（除了守护线程）都是平级关系,不像父子进程一样(父进程先消亡子变成孤儿进程)，只有除了守护线程外所有线程都结束了，才会结束JVM

> **孤儿进程**：一个父进程退出，而它的一个或多个子进程还在运行，那么那些子进程将成为孤儿进程。孤儿进程将被init进程(进程号为1)所收养，并由init进程对它们完成状态收集工作。
>
> **僵尸进程**：一个进程使用fork创建子进程，如果子进程退出，而父进程并没有调用wait或waitpid获取子进程的状态信息，那么子进程的进程描述符仍然保存在系统中。这种进程称之为僵死进程。

如果main方法中没有创建其他线程，那么当main方法返回时==>JVM就会结束==>Java应用程序。

但如果main方法中创建了其他线程，那么JVM就要在主线程和其他线程之间轮流切换，保证每个线程都有机会使用CPU资源，main方法返回(主线程结束)JVM也不会结束，要一直等到该程序所有线程全部结束才结束Java程序(另外一种情况是：程序中调用了Runtime类的exit方法，并且安全管理器允许退出操作发生。这时JVM也会结束该程序)。

那么又有个思考，JVM是怎么知道线程都结束的呢？

JVM中有一个线程DestroyJavaVM，执行main()的线程在main执行完后调用JNI中的jni_DestroyJavaVM()方法唤起DestroyJavaVM线程。JVM在Jboss服务器启动之后，就会唤起DestroyJavaVM线程，处于等待状态，等待其它线程（java线程和native线程）退出时通知它卸载JVM。线程退出时，都会判断自己当前是否是整个JVM中最后一个非deamon线程，如果是，则通知DestroyJavaVM线程卸载JVM。ps：扩展一下：1.如果线程退出时判断自己不为最后一个非deamon线程，那么调用thread->exit(false)，并在其中抛出thread_end事件，jvm不退出。2.如果线程退出时判断自己为最后一个非deamon线程，那么调用before_exit()方法，抛出两个事件： 事件1：thread_end线程结束事件、事件2：VM的death事件。然后调用thread->exit(true)方法，接下来把线程从active list卸下，删除线程等等一系列工作执行完成后，则通知正在等待的DestroyJavaVM线程执行卸载JVM操作。

***

***

***

### 参考链接

* [ThreadLocal](https://www.jianshu.com/p/3c5d7f09dfbd)

* [ThreadLocal详解](https://www.jianshu.com/p/3bb70ae81828)
* [InheritableThreadLocal详解](https://www.jianshu.com/p/94ba4a918ff5)
* [ThreadLocal-面试必问深度解析](https://www.jianshu.com/p/98b68c97df9b)
* [Java中的ThreadLocal详解](https://www.cnblogs.com/fsmly/p/11020641.html)
* [ThreadLocal会发生内存泄露吗？如何解决？](https://www.nowcoder.com/discuss/69456)

* [Java中的父线程与子线程](https://my.oschina.net/hosee/blog/509557)

* [孤儿进程与僵尸进程](https://www.cnblogs.com/Anker/p/3271773.html)
* [Thread的setDaemon(true)方法的作用](https://www.cnblogs.com/alter888/p/10490332.html)
* [Java多线程父子线程关系 多线程中篇（六）](https://www.cnblogs.com/noteless/p/10371174.html)

