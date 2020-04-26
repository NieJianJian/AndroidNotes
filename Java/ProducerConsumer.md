## 生产者-消费者模型

### 一. 前言

　　生产者消费者模式并不是GOF提出的23种设计模式之一，23种设计模式都是建立在面向对象的基础之上的，但其实**面向过程**的编程中也有很多高效的编程模式，生产者消费者模式便是其中之一，它是我们编程过程中最常用的一种设计模式。

> 面向过程就是分析出解决问题所需要的步骤，然后用函数把这些步骤一步一步实现，使用的时候一个一个依次调用就可以了。
>
> 面向对象是把构成问题事务分解成各个对象，建立对象的目的不是为了完成一个步骤，而是为了描叙某个事物在整个解决问题的步骤中的行为。

***

***

***

### 二. 简介

#### 1. 是什么

**生产者**：用来生产数据的模块。

**消费者**：用来处理数据的模块。

**缓冲区**：单单生产者和消费者还构不成生产者-消费者模式，生产者和消费者之间不直接通讯，而是将缓冲区作为中介。生产者将数据放入缓冲区，而消费者将缓冲区的数据取出，进行处理。

生产者消费者模式的大概结构如下图：

![图：生产者消费者模式示意图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/producerconsumer.jpeg)

　　为了不至于太抽象，举一个例子：一个饭店有很多厨师和服务员，厨师和服务员是不直接打交道的，而是厨师把菜做好后，放到窗口，服务员从窗口把菜直接端走给客人就好了。在这个过程中：

* 每一个厨师就相当于一个生产者；
* 每一个服务员就相当于一个消费者；
* 放菜的窗口就相当于缓冲区。

#### 2. 解决什么问题

　　生产者消费者模型问题是线程模型中的经典问题，在并发编程中使用生产者和消费者模式能够解决绝大多数并发问题。该模式通过平衡生产线程和消费线程的工作能力来提高程序的整体处理数据的速度。

　　在线程世界中，生产者就是生产数据的线程，消费者就是消费数据的线程。在多线程开发当中，如果生产者处理速度很快，而消费者处理速度很慢，那么生产者就必须等待消费者处理完，才能继续生产数据。同样的道理，如果消费者的处理能力大于生产者，那么消费者就必须等待生产者。为了解决这个问题于是引入了生产者和消费者模式。

**Q：为什么不让生产者直接调用消费者，而搞个缓冲区**？

**A：好处如下**：

* **解耦**：

  生产者和消费者都依赖于缓冲区，两者之间不直接依赖，耦合度降低了。

* **支持并发**：

  由于函数调用是同步的（或者阻塞的），在消费者还在处理，方法没有返回之前，生产者只好一直等待。

  当有了缓存区，生产者只需要把生产的数据往缓存去一丢，就可以直接去生产下一个数据了。

* **支持忙闲不均**：

  当生产者制造数据很快的时候，消费者来不及处理，这时候就可以将数据放入缓冲区，等什么时候消费者空闲了再来取走数据进行处理。

***

***

***

### 三. 生产者-消费者模型的实现方式

我们来假设：

* 厨师是生产者，服务员是消费者，窗口是缓冲区；
* 由于窗口能放的菜数量是有限的，我们假设只能放5个菜；
* 当厨师做完菜之后需要看一下窗口是不是满了，如果满了，就在旁边抽烟等待；
* 服务员取菜的时候，如果窗口是空的，就在旁边抽烟等待。

先来描述一下类——菜：

```java
public class Food {
    public static volatile int counter = 0; //代表生产的第几个菜
    public static final int MAX_COUNT = 100; // 假设一天做够100个菜，厨师也就该下班了
    public static final int MAX_QUEUE_SIZE = 5; // 窗口最多存放的菜的数量，也就是阻塞队列的大小。
    private int i;
    public Food() {
        i = ++counter;
    }
    @Override
    public String toString() {
        return "第" + i + "个菜";
    }
}
```

* 每次创建Food对象，i字段都+1，代表创建的第几道菜。
* MAX_COUNT代表最大菜品数，假设一天做够100到菜就该下班了。
* MAX_QUEUE_SIZE队列最大数，缓冲区队列的最大值，也就是窗口最多存放的菜的数量。

定义一个工具类：

```java
public class SleepUtil {
    private static Random random = new Random();
    public static void randomSleep() {
        try {
            Thread.sleep(random.nextInt(1000));
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

* 让当前线程随机休眠1秒以内的时间，模拟厨师炒菜的耗时和服务员传菜的耗时。

接下来，创建生产者（Cook：厨师类）和消费者（Waiter：服务员类），并且通过不同的方法来实现生产者消费者模式。

#### 3.1 使用wait()和notify()方法实现

```java
public class Cook extends Thread {
    private Queue<Food> queue;
    public Cook(Queue<Food> queue, String name) {
        super(name);
        this.queue = queue;
    }
    @Override
    public void run() {
        while (true) {
            Food food;
            SleepUtil.randomSleep();    //模拟厨师炒菜时间
            if (Food.counter >= Food.MAX_COUNT) {
                System.out.println("今天已经做完 " + Food.counter + " 道菜了，"
                        + getName() + "下班！");
                return;
            } else {
                food = new Food();
            }
            System.out.println(getName() + " 生产了" + food);
            synchronized (queue) {
                while (queue.size() >= Food.MAX_QUEUE_SIZE) {
                    try {
                        System.out.println("队列元素超过5个，为：" + queue.size()
                                + " " + getName() + "抽根烟等待中");
                        queue.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                queue.add(food);
                queue.notifyAll();
            }
        }
    }
}
```

```java
public class Waiter extends Thread {
    private Queue<Food> queue;
    public Waiter(Queue<Food> queue, String name) {
        super(name);
        this.queue = queue;
    }
    @Override
    public void run() {
        while (true) {
            Food food;
            synchronized (queue) {
                while (queue.size() < 1) {
                    if (Food.counter >= Food.MAX_COUNT) {
                        System.out.println("今天已经做完 " + Food.counter +
                                " 道菜了，并且窗口的菜也传完了，"+ getName() + "下班！");
                        return;
                    }
                    try {
                        System.out.println("队列元素个数为：" + queue.size() +
                                "，" + getName() + "抽根烟等待中");
                        queue.wait();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                food = queue.remove();
                System.out.println(getName() + " 获取到：" + food);
                queue.notifyAll();
            }
            SleepUtil.randomSleep();    //模拟服务员端菜时间
        }
    }
}
```

定义一个Restaurant类，代表餐馆真实发生的事情：

```java
public class Restaurant {
    public static void main(String[] args) {
        Queue<Food> queue = new LinkedList<>();
        new Cook(queue, "1号厨师").start();
        new Cook(queue, "2号厨师").start();
        new Cook(queue, "3号厨师").start();
        new Waiter(queue, "1号服务员").start();
        new Waiter(queue, "2号服务员").start();
        new Waiter(queue, "3号服务员").start();
    }
}
```

* 创建3个厨师和3个服务员，并且内部维护同一个queue队列，代表缓冲区。

#### 3.2 使用ReentrantLock实现

首先定义一个LockUtil工具类，用来声明Lock和相应Condition对象：

```java
public class LockUtil {
    private Lock mLock = new ReentrantLock();    // 创建一个锁对象
    private Condition notFull = mLock.newCondition();    // 缓冲区非满的条件变量
    private Condition notEmpty = mLock.newCondition();    // 缓冲区非空的条件变量
    public Lock getLock() { return mLock; }
    public Condition getNotFull() { return notFull; }
    public Condition getNotEmpty() { return notEmpty; }
}

```

接下来定义Cook类和Waiter类

```java
public class Cook extends Thread {
    private Queue<Food> queue;
    private LockUtil mLockUtil;
    public Cook1(Queue<Food> queue, String name, LockUtil lockUtil) {
        super(name);
        this.queue = queue;
        this.mLockUtil = lockUtil;
    }
    @Override
    public void run() {
        while (true) {
            Food food;
            SleepUtil.randomSleep();    //模拟厨师炒菜时间
            if (Food.counter >= Food.MAX_COUNT) {
                System.out.println("今天已经做完 " + Food.counter + " 道菜了，"
                        + getName() + "下班！");
                return;
            } else {
                food = new Food();
            }
            System.out.println(getName() + " 生产了" + food);
            mLockUtil.getLock().lock();
            try {
                while (queue.size() >= Food.MAX_QUEUE_SIZE) {
                    try {
                        System.out.println("队列元素超过5个，为：" + queue.size()
                                + " " + getName() + "抽根烟等待中");
                        mLockUtil.getNotFull().await();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                queue.add(food);
                mLockUtil.getNotEmpty().signalAll();
            } finally {
                mLockUtil.getLock().unlock();
            }
        }
    }
}
```

```java
package com.nj.test.producer;

import java.util.Queue;

/**
 * Author: NieJ
 * Created by niejianjian on 2020/4/26.
 * 说明：
 */
public class Waiter extends Thread {
    private Queue<Food> queue;
    private LockUtil mLockUtil;
    public Waiter1(Queue<Food> queue, String name, LockUtil lockUtil) {
        super(name);
        this.queue = queue;
        this.mLockUtil = lockUtil;
    }
    @Override
    public void run() {
        while (true) {
            Food food;
            mLockUtil.getLock().lock();
            try {
                while (queue.size() < 1) {
                    if (Food.counter >= Food.MAX_COUNT) {
                        System.out.println("今天已经做完 " + Food.counter +
                                " 道菜了，并且窗口的菜也传完了，" + getName() + "下班！");
                        return;
                    }
                    try {
                        System.out.println("队列元素个数为：" + queue.size() +
                                "，" + getName() + "抽根烟等待中");
                        mLockUtil.getNotEmpty().await();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
                food = queue.remove();
                System.out.println(getName() + " 获取到：" + food);
                mLockUtil.getNotFull().signalAll();
            } finally {
                mLockUtil.getLock().unlock();
            }
            SleepUtil.randomSleep();    //模拟服务员端菜时间
        }
    }
}
```

接下来定义Restaurant类，代表餐馆真实发生的事情：

```java
public class Restaurant {
    public static void main(String[] args) {
        LockUtil lockUtil = new LockUtil();
        Queue<Food> queue = new LinkedList<>();
        new Cook(queue, "1号厨师", lockUtil).start();
        new Cook(queue, "2号厨师", lockUtil).start();
        new Cook(queue, "3号厨师", lockUtil).start();
        new Waiter(queue, "1号服务员", lockUtil).start();
        new Waiter(queue, "2号服务员", lockUtil).start();
        new Waiter(queue, "3号服务员", lockUtil).start();
    }
}
```

* 将生成的LockUtil对象传入到Cook和Waiter中进行使用。

#### 3.3 使用BlockingQueue实现

* put(e) ： 将元素e加到BlockingQueue里，如果不能容纳，则调用此方法的线程被阻塞直到BlockingQueue里面有空间再继续添加 。
*  take() ： 取走BlockingQueue里排在队首的对象，若BlockingQueue为空，则进入等待状态直到BlockingQueue有新的对象被加入为止。

接下来创建Cook类和Waiter类

```java
public class Cook extends Thread {
    private BlockingQueue<Food> queue;
    public Cook2(BlockingQueue<Food> queue, String name) {
        super(name);
        this.queue = queue;
    }
    @Override
    public void run() {
        while (true) {
            Food food;
            SleepUtil.randomSleep();    //模拟厨师炒菜时间
            if (Food.counter >= Food.MAX_COUNT) {
                System.out.println("今天已经做完 " + Food.counter + " 道菜了，"
                        + getName() + "下班！");
                return;
            } else {
                food = new Food();
            }
            System.out.println(getName() + " 生产了" + food);
            try {
                queue.put(food);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
public class Waiter extends Thread {
    private BlockingQueue<Food> queue;
    public Waiter2(BlockingQueue<Food> queue, String name) {
        super(name);
        this.queue = queue;
    }
    @Override
    public void run() {
        while (true) {
            Food food;
            try {
                if (Food.counter >= Food.MAX_COUNT && queue.size() <= 0) {
                    System.out.println(getName() + "传完了所有菜");
                    return;
                }
                food = queue.take();
                System.out.println(getName() + " 获取到：" + food);
                if (Food.counter >= Food.MAX_COUNT && queue.size() <= 0) {
                    System.out.println(getName() + "传完了所有菜");
                    return;
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            SleepUtil.randomSleep();    //模拟服务员端菜时间
        }
    }
}
```

* Waiter类中有两次同样的判断，第二次，是为了保证当前Waiter对象调用了take方法，返回了最后一个Food对象，处理完之后，结束当前Waiter线程对象；第一次判断，是为了防止其他Waiter线程对象调用到take方法时，由于已经没有了元素，造成了阻塞，无法停止，根据相关判断，直接return。

接下来定义Restaurant类，代表餐馆真实发生的事情：

```java
public class Restaurant {
    public static void main(String[] args) {
        BlockingQueue<Food> queue = new ArrayBlockingQueue<Food>(5);
        new Cook(queue, "1号厨师").start();
        new Cook(queue, "2号厨师").start();
        new Cook(queue, "3号厨师").start();
        new Waiter(queue, "1号服务员").start();
        new Waiter(queue, "2号服务员").start();
        new Waiter(queue, "3号服务员").start();
    }
}
```

* 设置阻塞队列的大小为5。

***

***

***

### 参考链接

* [生产者/消费者模式的理解及实现](https://blog.csdn.net/u011109589/article/details/80519863)
* [聊聊并发——生产者消费者模式](https://www.infoq.cn/article/producers-and-consumers-mode/?amp%3butm_medium=popular_links_homepage)
* [你真的理解生产者/消费者模式吗？](https://cloud.tencent.com/developer/article/1458626)
* [Java生产者和消费者模型的5种实现方式](https://www.jianshu.com/p/66e8b5ab27f6)
* [生产者-消费者模型的三种实现方式](https://www.cnblogs.com/twoheads/p/10137263.html)

