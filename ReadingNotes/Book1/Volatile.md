## 对于volatile型变量的特殊规则

关键字volatiile可以说是Java虚拟机提供的最轻量级的同步机制。

当一个变量定义为volatile之后，将具备两种特性：

* **第一是保证此变量对所有线程的可见性**，这里的可见性是指当一条线程修改了变量的值，新值对于其他线程来说是立即得知的。

  　　volatile变量在各个线程的工作内存中不存在一致性问题（在各个线程的工作内存中，volatile变量也可以存在不一致的情况，但由于每次使用之前都要先刷新，执行引擎看不到不一致的情况，因此可以认为不存在不一致的问题），但是Java里面的运算并**非原子操作**，导致volatile变量的运算在并发下一样是不安全的。通过代码来说明原因，代码如下：

  ```java
  /**
   * volatile变量自增运算测试
   */
  public class VolatileTest {
      public static volatile int race = 0;
      public static void increase() {
          race++;
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

  　　上述代码每次运行结果，都是一个小于200000的数字，为什么呢？

  　　问题出现在自增运算"race++"之中，我们用Javap反编译这段代码，得到如下字节码内容：

  ```
  public static void increase();
      Code:
          stack=2, locals=0, args_size=0
          0: getstatic     #2                  // Field race:I
          3: iconst_1
          4: iadd
          5: putstatic     #2                  // Field race:I
          8: return
          LineNumberTable:
              line 9: 0
              line 10: 8
  ```

  　　上述代码可以看到，只有一行代码的increase()方法在Class文件中是由4条字节码指令构成的（return指令不是由race++产生的，这条指令可以不计算），从字节码层面上很容易就分析出并发失败的原因了：当getstatic指令把race的值渠道操作栈顶时，volatile关键字保证了race的值在此时是正确的，但是在执行iconst_1、iadd这些指令的时候，其他线程可能已经把race的值加大了，而操作栈顶的值就变成了过期的数据，所以putstatic指令执行后就可能把较小的值同步回主内存之中。

  　　由于volatile变量只能保证可见性，在不符合以下两条规则的运算场景中，我们仍然要通过加锁（使用synchronized或java.util.current中的原子类）来保证原子性：

  * 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。

  * 变量不需要与其他的状态变量共同参与不变约束。

    而像如下代码所示的这类场景就很适合volatile变量来控制并发：

  ```java
  volatile boolean shutdownRequested;
  public void shutdown() {
      shutdownRequested = true;
  }
  public void doWork() {
      while (!shutdownRequested) {
          // do stuff
      }
  }
  ```

* **第二是禁止指令重排序优化**，普通的变量仅仅会保证在该方法执行过程中所有依赖赋值结果的地方都能获取到正确的结果，而不能保证变量赋值操作的顺序与程序代码中的执行顺序一致。因为一个线程的方法执行过程中无法感知到这点，这也就是Java内存模型中描述的"*线程内表现为串行的语义*"。通过下面的例子来看看为何指令重排序会干扰程序的并发执行：

  ```java
  Map configOptions;
  char[] configText;
  // 此变量必须定义为volatile
  volatile boolean initealized = false;
  
  // 假设以下代码在线程A中执行
  // 模拟读取配置信息，当读取完成后将initialized设置为true以通知其他线程配置可用
  configOptions=new HashMap();
  configText=readConfigFile(fileName);
  processConfigOptions(configText,configOptions);
  initealized=true;
  
  // 假设以下代码在线程B中执行
  // 等待initialized为true，代表线程 A已经配把配置信息初始化完成
  while(!initialized){
  sleep();
  }
  // 使用线程A中初始化好的配置信息
  doSomethingWithConfig();
  ```

  　　上述代码是一段伪代码。如果定义initialized变量时没有使用volatile修饰，就可能会由于指令重排序的优化，导致位于线程A中的最后一句代码"initialized=true"被提前执行（对应的汇编代码被提前执行），这样在线程B中使用配置信息的代码就可能出现错误，而volatile可以避免此类情况发生。

  　　**指令重排序**是指CPU采用了允许将多条指令不按规定的顺序分开发送给各相应电路单元处理。但并不是说指令任意重排，CPU需要能正确处理指令依赖情况以保障程序能得出正确的执行结果。譬如指令1把地址A中的值加10，指令2把地址A中的值乘以2，指令3把地址B中的值减去3，这时指令1和指令2是有依赖的，它们的顺序不能重排，但指令3可以可以重排到1、2之前或者中间，只要保证CPU执行后面依赖到A、B值的操作时能获取到正确的A和B值即可。

  　　假定T表示一个线程，V和W分别表示两个volatile变量，那么在进行read、laod、use、assign、store和write操作时需要满足如下规则：

  * 只有当线程T对变量V执行的前一个动作是load时，线程T才能对变量V执行use操作；并且只有当线程T对变量V执行的后一个动作是use时，线程T才能对变量V执行load动作。线程T对变量V的use动作可以认为时和线程T对变量V的load、read动作相关联，必须连续一起出现（**这条规则要求在工作内存中，每次使用V前都必须先从主内存刷新最新的值，用于保证能看见其他线程对变量V所做修改后的值**）。
  * 只有当线程T对变量V执行的前一个动作是assign时，线程T才能对变量V执行store动作；并且，只有当线程T对变量V执行的后一个动作是store时，线程T才能对变量V执行assign动作。线程T对变量V的assign动作可以认为是和线程T对变量V的store、write动作相关联，必须连续一起出现（**这条规则要求在工作内存中，每次修改V后都必须立刻同步回主内存，用于保证其他线程可以看到自己对变量V所做的修改**）。
  * 假定动作A是线程T对变量V实施的use或assign动作，假定动作F是和动作A相关联的load或store动作，假定动作P是和动作F相应的对变量V的read或write动作；类似的，假定动作B是线程T对变量W实施的use或assign动作，假定动作G是和动作B相关联的load或store动作，假定动作Q是和动作G相应的对变量W的read或write操作。如果A先于B，那么P先于Q（**这条规则要求volatile修饰的变量不会被指令重排序优化，保证代码执行顺序下与程序顺序相同**）。

使用 volatile 后可以通过 cpu 指令屏障强制要求读操作发生在写操作之后，并且其他线程在读取该共享变量时，需要先清理自己的工作内存的该值，转而重新从主存读取，volatile 保证一定会刷新，但是不写也不一定其他线程看不见。