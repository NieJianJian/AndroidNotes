## 线程安全与锁优化

### 1 线程安全

　　**定义：**当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象就是线程安全的。

### 1.1 Java语言中的线程安全

按照线程安全的"安全程度"由强至弱来排序，可以将Java语言中各种操作共享的数据分为以下5类：不可变、绝对线程安全、相对线程安全、线程兼容和线程对立。

* **1.不可变**

  　　不可变的对象一定是线程安全的，无需采取任何的线程安全保障措施，如final关键字带来的可见性，只要一个不可变的对象被正确的构建出来（没有发生this引用逃逸的情况），那其外部的可见状态永远不会改变，永远也不会看到它在多个线程之中处于不一致的状况。

  * 如果共享数据是一个基本数据类型，只要在定义时使用final关键字保证它是不可变的。
  * 如果共享数据是一个对象，那就需要保证对象的行为不会对其他状态产生任何影响才行。如java.lang.String类的对象，它是一个典型的不可变对象，我们调用它的substring()、replace()、concat()这些方法都不会影响它原来的只，只会返回一个新构造的字符串对象。

  　　保证对象行为不影响自己状态的途径有很多，其中最简单的就是把对象中带有状态的变量都声明为final，这样在构造函数结束之后，他就是不可变的。 如java.lang.Integer中，它通过将内部状态变量value定义为final来保障状态不变。

  　　Java API中符合不可变要求的类型，除了**String**之外，常用的还有**枚举类型**，以及java.lang.Number的部分子类，如**Long**和**Double**等数值包装类型，**BigInteger**和**BigDecimal**等大数据类型。

* **2.绝对线程安全**

  　　Java API中标注自己是线程安全的类，都不是绝对的线程安全。例如java.util.Vector是一个线程安全的容器，因为它的add()、get()和size()、这类方法都是被synchronized修饰的，尽管这样效率低，但确实是安全的。但是，即使它所有的方法都被修饰成同步，也不意味着调用它的时候永远都不再需要同步手段，如下测试代码（**对Vector线程安全的测试**）：

  ```java
  private static Vector<Integer> vector = new Vector<>();
  public static void main(String[] args) {
      for (int i = 0; i < 10; i++) {
          vector.add(i);
      }
      Thread removedThread = new Thread(new Runnable() {
          @Override
          public void run() {
              for (int i = 0; i < vector.size(); i++) {
                  vector.remove(i);
              }
          }
      });
      Thread printThread = new Thread(new Runnable() {
          @Override
          public void run() {
              for (int i = 0; i < vector.size(); i++) {
                  System.out.println(vector.get(i));
              }
          }
      });
      removedThread.start();
      printThread.start();
      // 不要同时产生过多的线程，否则会导致操作系统假死
      while (Thread.activeCount() > 20) ;
  }
  ```

  运行结果报错：`ArrayIndexOutOfBoundsException`。报错是因为如果另一个线程恰好在错误的时间里删除了一个元素，导致序号i已经不再可用的话，再用i访问数组就会报错。为了保证代码正确执行下去，改为如下代码：

  ```java
  Thread removedThread = new Thread(new Runnable() {
      @Override
      public void run() {
          synchronized (vector) {
              for (int i = 0; i < vector.size(); i++) {
                  vector.remove(i);
              }
          }
      }
  });
  Thread printThread = new Thread(new Runnable() {
      @Override
      public void run() {
          synchronized (vector) {
              for (int i = 0; i < vector.size(); i++) {
                  System.out.println(vector.get(i));
              }
          }
      }
  });
  ```

* **3.相对线程安全**

  　　相对的线程安全就是我们通常意义上所讲的线程安全，它需要保证对这个对象单独的操作是线程安全的。大部分的线程安全类都属于这种类型，例如Vector、HashTable、Collections的synchronizedCollection()方法包装的集合等。

* **4.线程兼容**

  　　线程兼容是指对象本身并不是线程安全的，我们平常说一个类不是线程安全的，绝大多数是指这一种情况。Java API中大部分类都属于线程兼容，如与前面Vector和HashTable相对应的集合类ArrayList和HashMap等。

* **5.线程对立**

  　　线程对立是指无论调用端是否才去了同步措施，都无法在多线程环境中并发使用的代码。很少出现。如Thread类的suspend()和resume()方法，如果连个线程同时持有一个线程对象，一个尝试去中断线程，另一个尝试去恢复线程，如果并发进行的话，无论调用时是否进行了同步，目标线程都是存在死锁风险的，如果suspend()中断的线程就是即将要执行resume()的那个线程，那就肯定要产生死锁了。常见的线程对立操作还有System.setIn()、System.setOut()和System.runFinalizersOnExit()等。

### 1.2 线程安全的实现方法

* **1.互斥同步**

  　　*同步*是指多个线程并发访问共享数据时，保证共享数据在同一时刻只被一个（或者是一些，使用信号量的时候）线程使用。而互斥时实现同步的一种手段，临界区、互斥量和信号量都是主要的互斥实现方式。因此，互斥是因，同步是果；互斥是方法，同步是目的。

  　　Java中**最基本的互斥同步手段就是synchronized关键字**。synchronized关键字经过编译之后，会在同步块的前后分别行程monitorenter和monitorexit这两个字节码指令，这两个字节码都需要一个reference类型的参数来指明要锁定和解锁的对象。

  * 如果Java对象程序中的synchronized明确指定了对象参数，那就是这个对象的reference
  * 如果没有明确指定，那就根据synchronized修饰的是实例方法还是类方法，去取对应的对象实例或Class对象来作为锁对象。

  　　在执行monitorenter指令时，首先要尝试获取对象的锁。如果这个对象没被锁定，或者当前线程已经拥有了那个对象的锁，把锁的计数器加1，相应的，在执行monitorexit指令时会将锁计数器减1，当计数器为0时，锁就被释放。如果获取对象锁失败，那当前线程就要阻塞等待，知道对象锁被另外一个线程释放为止。

  　　synchronized同步块对同一条现成来说是可重入的，不会出现把自己把自己锁死的问题。同步块在已进入的线程执行完成之前，会阻塞后面其他线程的进入。Java的线程是映射到操作系统的原生线程上的，如果要阻塞或唤醒一个现成，都需要操作系统来帮忙完成，这就需要从用户态切换到核心态中，因此状态转换需要耗费很多的处理器时间。对于代码简单的同步块（如synchronized修饰的getter()或setter()方法），状态转换消耗的时间可能比用户代码执行的时间都要长。所以**Synchronized是Java语言中一个重量级的操作。**而虚拟机本身也会进行一些优化，如在通知操作系统阻塞线程之前加入一段自旋等待过程，避免频繁地切入到核心态之中。

  　　除了synchronized之外，还可以使用java.util.concurrent包中的重入锁（ReentrantLock）来实现同步，基本语法上，ReentrantLock与synchronized很相似，它们都具备一样的线程重入特性，只是代码写法上有点区别，一个表现为API层面的互斥锁（lock()和unlock()方法配合try/finally语句块来完成），另一个表现为原生语法层面的互斥锁。ReetrantLock增加了以下3项高级功能：

  * **等待可中断**是指持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待。可中断特性对处理执行时间非常长的同步块很有帮助。
  * **公平锁**是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来一次获得锁；而非公平锁则不保证这一点，在锁被释放时，任何一个等待锁的线程都有机会获得锁。synchronized中的锁时非公平的，ReentrantLock默认情况下也是非公平的，但可以通过带布尔值的构造函数要求使用公平锁。
  * **锁绑定多个条件**是指一个ReentranLock对象可以同时绑定多个Condition对象，而在synchronized中，锁对象的wait()和notify()或notifyAll()方法可以实现一个隐含的条件，如果要和多于一个条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则无须这样做，只需要多次调用newCondition()方法即可。

* **2.非阻塞同步**

  * **阻塞同步**：互斥同步最主要的问题就是进行线程阻塞和唤醒所带来的性能问题，所以也称阻塞同步。
    * 悲观的并发策略——认为只要不做同步措施，就会出问题
  * **非阻塞同步**：实现的时候不需要主动把线程挂起。
    * 乐观的并发策略——先操作，如果共享数据有冲突，再采取补偿措施（如不断重试，直到成功）

  　　乐观的并发策略需要"硬件指令集的发展"才能进行，因为我们需要操作和冲突检测这两个步骤具备原子性。硬件保证一个从语义上看起来需要多次操作的行为只通过一条处理器指令就能完成，常用指令有：

  * 测试并设置

  * 获取并添加

  * 交换

  * 比较并叫唤（CAS）

    　　CAS指令需要有3个操作数，分别时内存位置（可理解为变量的内存地址，用V表示）、旧的预期值（用A表示）和新值（用B表示）。CAS指令执行时，当前仅当V符合旧预期值A时，处理器用新值B更新V得值，否则它就不执行更新，但是无论是否更新V的值，都会返回V的旧值，上述的操作时原子操作。

  * 加载链接 / 条件存储（LL / SC）

  　　CAS操作由sun.misc.Unsafe类里面的compareAndSwapInt()和compareAndSwapLong()等几个方法包装提供，虚拟机在内部对这些方法做了特殊处理，即时编译出来的结果就是一条平台相关的处理器CAS指令，没有方法调用的过程，或者可以认为时无条件内联进去了（这种被虚拟机特殊处理的方法称为**固有函数**，类似的还有Math.sin()等）。

  　　由于Unsafe类不实提供给用户程序调用的类（Unsafe.getUnsafe()的代码中限制了只有启动类加载器加载的Class才能访问），因此，如果不采用反射的话，只能通过其他的Java API来间接使用，如J.U.C包里面的整数原子类，其中compareAndSet()和getAndIncrement()等方法都使用了Unsafe类的CAS操作。

  　　之前在[volatile](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book1/Volatile.md)中有一段自增的代码来证明volatile变量不具备原子性，如果改成如下代码，效率将会提高许多：

  ```java
  /**
   * Atomic变量自增运算测试
   */
  public class AtomicTest {
      public static AtomicInteger race = new AtomicInteger(0);
      public static void increase() {
          race.incrementAndGet(); // 原子操作
      }
      public static final int THREADS_COUNT = 20;
      public static void main(String[] args) {
          Thread[] threads = new Thread[THREADS_COUNT];
          for (int i = 0; i < THREADS_COUNT; i++) {
              threads[i] = new Thread(new Runnable() {
                  @Override
                  public void run() {
                      for (int i = 0; i < 10000; i++) {
                          increase();
                      }
                  }
              });
              threads[i].start();
          }
          // 等待所有累加线程都结束
          while (Thread.activeCount() > 1)
              Thread.yield();
          System.out.println(race);
      }
  }
  ```

  　　使用AtomicInteger代替int后，程序输出了正确的寄过，一切都要归功于incrementAndGet(0方法的原子性。它的实现其实非常简单，如下代码：

  ```java
  /**
   * Atomically increments by one the current value.
   * @return the updated value
   */
  public final int incrementAndGet() {
      for (;;){
          int current = get();
          int next = currrent + 1;
          if (compareAndSet(current, next)) 
              return next;
      }
  }
  ```

  　　incrementAndGet()方法在无限循环中，不断尝试将一个比当前值大1的新值赋给自己。如果失败了，那就说明在执行"获取-设置"操作的时候值已经有了修改，于是再次循环进行下一次操作，直到设置成功为止。

* **3.无同步方案**

  有一些代码天生就是线程安全的。比如以下两类：

  * **可重入代码**：也叫做纯代码，可以在代码执行的任何时刻中断它，转而去执行另外一段代码，而在控制权返回后，原来的程序不会出现任何错误。
    * *特征*：不依赖存储在堆上的数据和公用的系统资源、用到的状态量都由参数中传入、不调用非可重入的方法等。
    * *判断依据*：如果一个方法它的返回结果是可以预测的，只要输入相同的数据，就都能返回相同的结果，那它就满足可重入性的要求，当然也就是线程安全的。
  * **线程本地存储**：如果能保证一段代码中需要与其他代码共享的数据在同一线程中，这样就可以把共享数据的可见范围限制在同一个现成中，也就无需同步了。
    * *案例*：比如大部分使用消费队列的结构模式（如"生产者—消费者"模式）都会将产品的消费过程尽量在一个线程中消费完，其中最重要的应用案例就是经典Web交互模型中的"一个请求对应一个服务器线程"的处理方式。
    * 如果一个变量要被某个线程独享，可以通过java.lang.ThreadLocal类来实现线程本地存储。每一个线程的Thread对象中都有一个ThreadLocalMap对象，这个对象存储了一组以ThreadLocal.threadLocalHashCode为键，以本地线程变量为值的K-V值对，ThreadLocal对象就是当前线程的ThreadLocalMap的访问入口，每一个ThreadLocal对象都包含一个独一无二的threadLocalHashCode值，使用这个值就可以在线程K-V值对中找到对应的本地线程变量。

### 1.3 锁优化

* **1.自旋锁与自适应自旋**

  **起因：**互斥同步造成阻塞，挂起和恢复线程都需要转入内核态完成，性能压力大；共享数据的锁只会持续很短的时间，为了这段时间去挂起和恢复线程不值得。

  **自旋锁**：如果物理机器有一个以上的处理器，能让两个或以上的线程同时并行执行，我们就可以让后面请求锁的那个线程"稍等一下"，但不放弃处理器的执行时间，看看持有锁的线程是否很快就会释放锁，为了让线程等待，我们只需要让线程执行一个忙循环（自旋），这项技术就是自旋锁。

  　　自旋等待不能代替阻塞，虽避免线程切换开销，但占用处理器时间。锁占用时间越短，自旋效果越好，反之只会浪费处理器资源。所以自旋次数*默认值是10次*，超过限定次数就使用传统方式去挂起线程。

  **自适应自旋：**自适应意味着自旋的时间不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而它将允许自旋等待持续相对更长的时间，比如100次循环。另外，如果对于某个锁，自旋很少成功获得过，那在以后要获取这个锁时将可能省略掉自旋的过程，以避免浪费处理器资源。

* **2.锁消除**

  　　锁消除的主要判断依据来源于逃逸分析的数据支持，如果判断在一段代码中，堆上所有数据都不会逃逸出去从而被其他线程访问大，那就可以把它们当做栈上数据对待，不同加锁就可以进行消除了。

  　　程序员很清楚数据是否存在争用，但是有些同步措施并不是程序员自己加入的，如下代码：

  ```java
  public String concatString(Stirng s1, String s2, String s3) {
      return s1 + s2 + s3;
  }
  ```

  　　String是一个不可变的类，对字符串的连接操作总是通过生成新的String对象来进行的，因此Javac编译器会对String连接做自动优化，JDK 1.5以前转化为StringBuffer对象的连续append()操作，JDK 1.5以后转化为StringBuilder对象的连续append()操作。所以上述代码会变为如下代码：

  ```java
  public String concatString(Stirng s1, String s2, String s3) {
      StringBuffer sb = new StringBuffer();
      sb.append(s1);
      sb.append(s2);
      sb.append(s3);
      return sb.toString();
  }
  ```

  　　每个StringBuffer.append()方法中都有一个同步块，锁就是sb对象。由于变量sb的动态作用域被限制在concatString()方法内部，也就是说sb的所有引用永远不会逃逸到concatString()方法外，所以说可以消除掉。

* **3.锁粗化**

  　　原则上，同步块的作用范围尽量小，这样阻塞时间也就会短。但是，如上述代码，每一次append()方法都对sb进行加锁和解锁，将会导致不必要的性能损耗。

  　　所以，锁粗化就是将一串零碎的加锁操作，范围扩展到整个操作序列的外部，如上代码，就是扩展到第一个append()操作之前直至最后一个append()操作之后，这样只需要加锁一次就可以了。

* **4.轻量级锁**

  　　它名字中的"轻量级"时相对于使用操作系统互斥量来实现的传统所而言的。因此传统的锁机制也称为"重量级"锁。轻量级锁并不是用来代替重量级锁的，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。

  　　HotSpot虚拟机的对象的对象头部分的Mark Word（详见[Chapter2](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book1/Chapter2.md)中"对象的内存布局部分"）是实现轻量级锁和偏向锁的关键。在代码进入同步块的时候，如果此同步对象没有被锁定（锁标志位为"01"），虚拟机将在当前线程的栈帧中建立一个名为锁记录的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了一个Displaced前缀），这时线程堆栈与对象头的状态如下图：

  ![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/lightweightlocking_cas_before.jpg)

  　　然后虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针，如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位将变为"00"，即表示此对象处于轻量级锁定状态，这时候线程堆栈与对象头的状态如下图：

  <img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/lightweightlocking_cas_after.jpg" style="zoom:75%;" />

  　　如果这个更新操作失败了，虚拟机将会检查对象的Mark Word是否指向当前线程的栈帧，如果只说明当前线程已经拥有了这个对象的锁，那就直接进入同步块继续执行，否则说明这个锁对象已经被其他线程抢占了。如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要膨胀为重量级锁，锁标志的状态值变为"10"，Mark Word中存储的就是指向重量锁（互斥量）的指针，后面等待锁的进程也要进入阻塞状态。

  　　上面描述的是轻量级锁的加锁过程，解锁过程也是通过CAS，如果对象的Mard Word仍然指向着线程的锁记录，那就用CAS操作把对象当前的Mard Word和线程中复制的Displaced Mark Word替换回来，如果替换成功，整个同步过程就完成了。如果替换失败，说明有其他线程尝试过获取该锁，那就要在释放锁的同时，唤醒被挂起的线程。

  　　如果没有竞争，轻量级锁使用CAS操作避免了使用互斥量的开销，但如果存在锁竞争，除了互斥量的开销外，还额外发生了CAS操作，因此在有竞争的情况下，轻量级锁会比传统的重量级锁更慢。

* **5.偏向锁**

  　　偏向锁的目的是消除数据在无竞争情况下的同步原语，进一步提高程序的运行性能。如果说轻量级锁是在无竞争的情况下使用CAS操作去消除同步使用的互斥量，那偏向锁就是在无竞争的情况下把整个同步都消除掉，连CAS操作都不做了。

  　　偏向锁会偏向于第一个获得它的线程，如果在接下来的执行过程中，该锁没有被其他的线程获取，则持有偏向锁的线程将永远不需要进行同步。

  　　假设当前虚拟机启用了偏向锁，那么，当锁对象第一个被线程获取的时候，虚拟机将会把对象头的标志位设为"01"，即偏向模式。同时使用CAS操作把获取到这个锁的线程ID记录到对象的Mark Word中，如果CAS操作成功，持有偏向锁的线程以后每次进入到这个锁相关的同步块时，虚拟机都嗯可以不再进行任何同步操作（例如Locking、Unlocking及对Mark Word的Update等）。

  　　当有另外一个线程去尝试获取这个锁时，偏向模式就宣告结束。根据锁对象目前是否处于被锁定的状态，撤销偏向后恢复到未锁定或轻量级锁定的状态，后续的同步操作就如轻量级锁那样操作。偏向锁、轻量级锁的状转化及对象Mark Word的关系如下图：

  <img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/biased_lightweight_conversion.jpg" style="zoom:80%;" />

  　　偏向锁可以提高带有同步但无竞争的程序性能。