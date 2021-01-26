## 第3章 Java内存模型

### 1.  Java内存模型的基础

#### 1.1 并发编程模型的两个关键问题

* 如何通信

  * 共享内存

    线程之间共享程序的公共状态，通过写-读内存中的公共状态进行隐式通信。

  * 消息传递

    线程之间没有公共状态，线程之间必须通过发送消息来显式进行通信。

* 如何同步

  共享内存并发模型中，同步是显式进行的，程序员必须显式指定某个方法或某段代码需要在线程之间互斥执行。

  消息传递的并发模型里，同步是隐式执行的。

#### 1.2 从源代码到指令序列的重排序

重排序分为3种：

* 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
* 指令级并行的重排序。现代处理器采用指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
* 内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是乱序执行。

从Java源代码到最终实际执行的指令序列，会经历下面3中重排序：

> 源代码 -> 1:编译器优化重排序 -> 2:指令级并行重排序 -> 3:内存系统重排序 -> 最终执行的指令序列

1属于编译器重排序，2、3术语处理器重排序。

* 对于编译器，JMM的编译器重排序规则会禁止特定类型的编译器重排序（不是所有的编译器重排序都要禁止）。
* 对于处理器重排序，JMM的处理器重排序规则会要求Java编译器在生成指令序列时，插入特定类型的内存屏障指令，通过内存屏障指令来禁止特定类型的处理器重排序。

为了保证内存可见性，Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序。

#### 1.3 happends-before简介

在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须**存在happens-before关系**。

与程序员密切相关的happens-before规则如下：

* 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
* 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
* volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
* 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

> 两个操作之间具有happens-before关系，并不意味着前一个操作必须要在后一个操作之前执行。happens-before仅仅要求前一个操作（执行结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。

***

### 2. 重排序

#### 2.1 数据依赖性

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。

* 写后读：`a = 1; b = a;`
* 写后写：`a = 1; a = 2;`
* 读后写：`a = b; b = 1;`

编译器和处理器在重排序时，会遵守**数据依赖性**，不会改变存在数据依赖关系的两个操作的执行顺序。

这里所说的数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作，不同处理器之间和不同线程之间的数据依赖性不被编译器和处理器考虑。

#### 2.2 as-if-serial语义

as-if-serial语义的意思是：不管怎么重排序，单线程的执行结果不能被改变。

```
double pi = 3.14;					// A
double r  = 1.0;					// B
double area = pi * r * r; // C
```

C不能排到A、B的前面，A和B可以重排序。

遵守as-if-serial语义的编译器、runtime和处理器共同为编写单线程程序的程序员创建一个幻觉：单线程程序是按顺序来执行的。

#### 2.3 重排序对多线程的影响

```
if (flag) {					// 1
		int i = a * a;  // 2
		...
}
```

操作1和操作2 存在控制关系，当代码中存在控制依赖性时，会影响指令序列执行的并行度。为此，编译器会采用**猜测**执行来克服控制相关性对并行度的影响。以处理器的猜测执行为例，执行线程B的处理器可以提前读取并计算`a*a`，然后把计算结果临时保存在一个名为**重排序缓冲**的硬件缓存中，当操作1的条件判断为真时，就把该计算结果写入变量i中。

猜测执行实质上对操作3和操作4做了重排序，单线程不会改变执行结果，多线程可能会改变执行结果。

***

### 3. 顺序一致性

如果程序是正确同步的，程序的执行将具有顺序一致性——即程序的执行结果与改程序在顺序一致性内存模型中的执行结果相同。

#### 3.1 顺序一致性内存模型

顺序一致性内存模型有两大特性：

1）一个线程中的所有操作必须按照程序的顺序来执行。

2）（不管程序是否同步）所有线程都只能看到一个单一的操作执行顺序。在顺序一致性内存模型中，每个操作都必须原子执行且立刻对所有线程可见。

顺序一致性模型有一个单一的全局内存，这个内存通过一个左右摆动的开关可以连接到任意一个线程，同时每一个线程必须按照程序的顺序来执行内存读/写操作。在任意时间点最多只能有一个线程可以连接到内存。当多个线程并发执行时，所有线程的所有内存读/写操作都被串行化（即在顺序一致性模型中，所有操作之间时全序关系）。

未同步程序在顺序一致性模型中虽然整体执行是无序的，但所有线程都只能看到一个一致的整体执行顺序。之所以能得到这个保证是因为顺序一致性内存模型中的每个操作都必须对任意线程可见。

但是JMM中没有这个保证。比如当前线程把写过的数据缓存到本地内存中，在没有刷新到祝内存之前，这个写操作仅对当前线程可见，只有当前线程把祝内存中写过的数据刷新到主内存之后，这个写操作才对其他线程可见。

#### 3.2 未同步程序的执行特性

线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值（0，Null，Flase），为了实现最小安全性，JVM在堆上分配内存时，首先会对内存进行清零，然后才会再上面分配对象（JVM内部会同步这两个操作）。因此，在已清零的内存空间分配对象时，域的默认初始化已经完成了。

在一些32位的处理器上，如果要求对64的long型和double型的写操作具有原子性，开销会比较大。JVM在这种处理器上运行时，可能会把一个64位的long型变量和double型变量的写操作拆分为两个32位的写操作，所以这个64位的写操作将不具有原子性。

***

### 4 volatile的特性

volatile特性

* 可见性。对一个volatile变量的读，总能看到（任意线程）对这个volatile变量最后的写入。
* 原子性。对任意单个volatile变量的读/写具有原子性，即使时64位的long型和double型变量，只要时volatile变量，都具有原子性。但类似于volatile++这种复合操作不具备原子性。

volatile的写-读与锁的释放-获取有相同的内存效果：

* volatile写和锁的释放有相同的内存语义；
* volatile读和锁的获取有相同的内存语义。

volatile的内存语义：

* 写：**当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存**。

* 读：**当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从祝内存读取共享变量**。

volatile写和volatile读的内存语义做个总结：

* 线程A写一个volatile变量，实质上是线程A向接下来将要读这个volatile变量的某个线程发出了（其对共享变量所做修改的）消息。
* 线程B读一个volatile变量，实质上是线程B接收了之前某个线程发出的（在写这个volatile变量之前对共享变量所做修改的）消息。
* 线程A写一个volatile变量，随后线程B读这个volatile变量，这个过程实质上是线程A通过主内存像线程B发送消息。

#### 4.1 volatile内存语义的实现

volatile重排序规则表

| 是否能重排序 | 第二个操作 | 第二个操作 | 第二个操作 |
| :----------: | :--------: | :--------: | :--------: |
|  第一个操作  | 普通读/写  | volatile读 | volatile写 |
|  普通读/写   |            |            |     NO     |
|  volatile读  |     NO     |     NO     |     NO     |
|  volatile写  |            |     NO     |     NO     |

* 当第二个操作是volatile写时，不管第一个操作是什么，都不能重排序。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。
* 当第一个操作是volatile读时，不管第二个操作是什么，都不能重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。
* 当第一个操作是volatile写，第二个操作是volatile读时，不能重排序。

**为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序**。

下面时基于保守策略的JMM内存屏障插入策略：

* 在每个volatile写操作的前面插入一个StoreStore屏障。
* 在每个volatile写操作的后面插入一个StoreLoad屏障。
* 在每个volatile读操作的后面插入一个LoadLoad屏障。
* 在每个volatile读操作的后面插入一个LoadStore屏障。

StoreStore屏障可以保证在volatile写之前，其前面的所有普通写操作已经对任意处理器可见了。这是因为StoreStore屏障将保障上面所有的普通写在volatile写之前都刷新到主内存。

volatile仅仅保证对单个volatile变量的读/写具有原子性，而锁的互斥执行的特性可以确保整个临界区代码的执行具有原子性。

***

### 5. 锁的内存语义

锁除了让临界区互斥执行外，还可以让释放锁的线程向获取同一个锁的线程发送消息。

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。

当线程获取锁时，JMM会把该线程对应的本地内存置为无效。从而使得被监视器保护的临界区代码必须从主内存中读取共享变量。

#### 5.1 ReentrantLock

ReentrantLock的实现依赖于Java同步框架`AbstractQueuedSynchronizer`。AQS使用一个整型的volatile变量（命名为state）来维护同步状态，volatile变量是ReentrantLock内存语义实现的关键

ReentrantLock分为公平锁和非公平锁。

* 公平锁

  * 加锁lock()调用轨迹如下

    1）ReentrantLock：lock()

    2）FairSync：lock()

    3）AbstractQueuedSynchronizer：acquire(int arg)

    4）ReentrantLock：tryAcquire(int acquires)

    第4步真正开始加锁，源代码如下：

    ```java
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();  									// 获取锁的开始，首先读取volatile变量state
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    ```

  * 解锁方法unlock()的调用轨迹如下

    1）ReentrantLock：unlock()

    2）AbstractQueuedSynchronizer：release(in arg)

    3）Sync：tryRelease(int releases)

    第3步真正开始释放锁，源代码如下：

    ```java
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);							// 释放锁的最后，写volatile变量state
        return free;
    }
    ```

  公平锁释放最后写volatile变量`state`，在获取锁时首先读这个volatile变量，根据volatile的happens-before规则，释放锁的线程在写volatile变量之前可见的共享变量，在获取锁的线程读取同一个volatile变量后将立即变得对获取锁的线程可见。

* 非公平锁

  * 非公平锁的释放和公平锁完全一样

  * 加锁方法lock()调用轨迹如下

    1）ReentrantLock：lock()

    2）NonfairSync：lock()

    3）AbstractQueuedSynchronizer：compareAndSetState(int expect, int update)

    第3步真正开始加锁，源代码如下：

    ```java
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    ```

    CAS方法：如果当前状态值等于预期值，则以原子方式将同步状态设置为给定的更新值。此操作具有volatile读写的内存语义。

* CAS

  CAS如何同时具有volatile读和volatile写的内存语义。分别从编译器和处理器角度来分析

  * 编译器

    编译器不会对volatile读和volatile读后面的任意内存操作重排序；编译器不会对volatile写和volatile写前面的任意内存操作重排序。结合这两个条件，意味着同时实现了volatile读和volatile写的内存语义，编译器不能对CAS与CAS前面和后面的任意内存操作重排序。

  * 处理器

    下面是sun.misc.Unsafe类的compareAndSwapInt()方法的源代码：

    ```java
    public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x);
    ```

    这个本地方法在openjdk中一次调用c++代码：unsafe.cpp、atomic.cpp、atomic_window_x86.inline.hpp。源代码如下：

    ```c++
    inline jint Atomic::cmpxchg (jint exchange_value, volatile jint* dest, jint compare_value){
        int mp = os::is_MP()
        __asm{
            mov edx, dest
            mov ecx, exchange_value
            mov eax, compare_value
            LOCK_IF_MP(mp)
            cmpxchg dword ptr [edx], ecx
        }
    }
    ```

    如上述源代码所示，程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序在多处理器上运行，就为cmpxchg指令加上lock前缀（Lock Cmpxchg）。

    intel手册对lock前缀的说明

    > 1）确保对内存的读 - 改 - 写 操作原子执行。旧的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存，开销大。新的处理器中，使用缓存锁定来保证指令执行的原子性，开销小。
    >
    > 2）禁止该指令，与之前和之后的读和写指令重排序
    >
    > 3）把写缓存区中的所有数据刷新到主内存

    上面第2点和第3点所具有的内存屏障效果，足以同时实现volatile读和volatile写的内存语义。

* 公平锁和非公平锁的内存语义的总结

  * 公平锁和非公平锁释放时，最后都要写一个volatile变量state
  * 公平锁获取时，首先会读volatile变量
  * 非公平锁获取时，首先会用CAS更新volatile变量，这个操作同时具有volatile读和volatile写的内存语义。

* 锁释放 - 获取的内存意义的实现至少有以下两种方式

  * 利用volatile变量的写 - 读所具有的内存语义
  * 利用CAS锁附带的volatile读和volatile写的内存语义

#### 5.2 concurrent包的实现

Java线程之间的通信有以下4种方式

* A线程写volatile变量，随后B线程读这个volatile变量
* A线程写volatile变量，随后B线程用CAS更新这个volatile变量
* A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量
* A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量。

concurrent包的通用化实现模式：

* 首先，声明共享变量volatile
* 然后，使用CAS的原子条件更新来实现线程之间的同步
* 同时，配合以volatile的读 / 写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），这些concurrent包中的基础类都是使用这种模式实现的，而concurrent包中的高层类，又是依赖于这些基础类来实现的。

***

### 6. final域的内存语义

### 6.1 final域的重排序规则

对于final域，编译器和处理器要遵循两个重排序规则

1）在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

2）初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。