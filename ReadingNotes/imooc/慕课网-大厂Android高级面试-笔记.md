### 1.面试准备

* 如何阅读招聘的要求

* 如何找准自身的定位

* 如何选择合适的岗位

* **如何准备对应的简历**

  要聚焦核心技能，不要到处熟练精通；要突出技术亮点，不要罗列开源框架；

  要体现业务背景，不要堆积项目细节；要明确项目成果，不要陈述项目过程。

## 第三章 Java技术

* **通用技术**

  * **1.Java的char是两个字节，如何存UTF-8字符？**

    字符集：Unicode、ASCII码。

    * JavaChar不存UTF-8的字节，而是UTF-16;
    * Unicode通用字符集占两个字节，例如"中";
    * Java String的length不是字符数；

  * **2.Java String可以有多长？**

    * 代码直接声明的String，是创建在栈中的，编译器就可以决定大小；从文件中读取的内容，是存在于堆中的，因为读取的内容大小不确定。
    * 最多存放65535个字节。
    * *.java文件中声明，长度为65535是，报错：java:constant string too long。 65534就ok。编译器bug导致，Gen.java中有<判断。
    * C语言的字符串的结尾是 \0，否则编译器不知道字符串的结尾。Java字符串是以对象的形式存在，已经拥有了length属性，所以不需要空字符结尾。
    * （栈）受字节码限制，字符串最终的MUTF-8字节数不超过65535；
    * （栈）Latin字符，受javac代码限制，最多65534个；
    * （栈）非Latin字符最终对应字节个数差异较大，最多字节数是65535；
    * （栈）如果运行时方法区设置过小，也会受到方法区大小的现在，导致内存溢出。
    * （堆）受虚拟机指令限制，字符数理论上限为Integer.MAX_VALUE，实际上可能会小于这个值。
    * （堆）如果堆内存较小，也可能会收到对内存的限制。

  * **3.java的匿名内部类有哪些限制？**
    
    * 匿名内部类的名字：外部类 + $N，N是匿名内部类的顺序。
    * Java10有类型推导机制，用var来声明变量。
    * 匿名内部类的构造方法：
      * 编译器生成
      * 参数列表包括
        * 外部对象（定义在非静态域内）
        * 父类的外部对象（父类非静态）
        * 父类的构造方法参数（父类有构造方法且参数列表不为空）
        * 外部捕获的变量（方法体内有引用外部final变量）
    * 只能继承一个父类或者实现一个接口；父类是非静态的类型，则需父类外部实例来初始化；如果定义在非静态作用域内，会引用外部类实例；只能捕获外部作用域内的final变量；创建时只有单一方法的接口可以用lambda转换，单一方法的抽象类不可以用lambda转换。 
  * **4.怎么样理解java的方法分派？**
    
    * 静态分派-方法重载分派
      * 编译期确定
      * 依据调用者的声明类型和方法参数类型
    * 动态分派-方法覆写分派
      * 运行时确定
      * 依据调用者在运行时的实际类型进行分派
    * 当存在子父类关系时，Java是根据声明的类型进行调用；Groovy是根据实际的类型进行调用。
  * **5.泛型的的实现机制是怎么样的？**
    
    * 类型擦除
      * 基本类型无法做为泛型实参，（装箱、拆箱）。SparseArray特意规避了装箱拆箱的过程。
      * 泛型类型无法用作方法重载。
      * 泛型类型无法用作真实的类型使用。
      * Gson.fromJson为什么要传入Class。
      * 静态方法无用引用类泛型参数。因为类的泛型参数只有在实例化的时候才能知道，静态方法调用不需要等到类实例化。静态方法可以自己声明泛型参数。
      * 类型强转的运行时开销。1.5以前使用需要手动强转，1.5开始，不需要手动强转，但是编译成字节码之后，还是需要强转。因为编译完之后类型擦除掉了。
    * 混淆时要保留签名信息：-keepattributes Signature。（混淆主要是针对字节码的处理）
    * Gson、Retrofit。
  * **6.Activity的onActivityResult为什么不设计成回调。**
    
    * 缺点
      * 代码处理逻辑分离，容易出现遗漏和不一致的问题
      * 写法不够直观，且结果数据没有类型安全保障，需要通过intent来传递。
      * 结果种类较多时，难以维护
    * Activity的销毁和恢复机制，不允许匿名内部类的出现。
    * Fragment的id可以重复，tag可以重复，fragment的id是container的id。fragment有一个字段mWho，可以标定唯一身份，可以通过反射拿到。

## 第四章 并发编程

* **1.如何停止一个线程？**

  * stop() ，已经被Deprecated，因为线程直接被停止掉，不安全。

  * 逻辑上来实现，将任务停止。

  * CPU、内存、文件等，都是线程共享的。

  * 为什么不能简单的停止一个线程？

    假如Thread1占用内存资源，并加了lock，Thread2想要访问内存资源，就会发生block，Thread1暂停之后，lock也不会释放，Thread2将会一直等待，如果此时Thread2也持有一个Thread1想要的锁，那就会发生死锁。如果Thread1被干掉，内存将会被立即释放，Thread2将会立即加锁，但是Trehad1写入资源写到一半，被干掉后，也没有机会清理内存。Thread2操作内存将会遇到一些问题，也可能遇到crash 。

  * 线程是协作的任务执行模式，任务完成，线程也会自动停止。让线程技术，其实就是让任务结束，

  * 中断方式

    * Interrupt
      * 抛异常进行处理
      * interrupted()方法判断进行处理。
    * boolean标志位

* **2.如何写出线程安全的程序？**

  * 什么是线程安全：可变资源（内存）线程间共享的问题。进程的内存是独享的，线程之间是共享进程内存的。

  * 如何实现线程安全。

    * 不共享资源

      ```java
      // 可重入函数
      public static int addTwo(int num) {
          return num + 2;
      }
      ```

      ThreadLocal。 

    * 共享不可变资源

    * 共享可变资源

      * 可见性 （volatile保证可见性，也可以禁止重排序）
        * 使用final关键字
        * 使用volatile关键字
        * 加锁，锁释放时会强制将缓存刷新到主内存。
      * 操作原子性
        * 加锁，保证操作的互斥性
        * 使用CAS指令（Unsafe. CompareAndSwapInt）
        * 使用原子数值类型（如AtomicInteger）
        * 使用原子属性更新器（AtomicReferenceFieldUpdater）
      * 禁止重排序 （final有禁止重排序的作用。）

* **3.ConcurrentHashMap如何实现并发访问？**

  * CHM的并发优化过程

    * JDK 5：分段锁，必要时加锁。（HashTable整个hash表加锁）

      通过高位运算，决定所处的segment位置，但是hash值较小的时候，对于3万多以内的，高位始终是15，所计算的segment，都集中于前面的位置，没办法均匀分布在不同的段，都扎堆了，这样就退化成了HashTable，所以这是个缺陷。

    * JDK 6：优化二次Hash算法。(single-word Wang/Jenkins hash)

    * JDK 7：段懒加载，volatile & cas。

      由于是段懒加载，用到的时候才会实例化，就会涉及到可见性的问题，所以大量使用volatile。

    * JDK 8：摒弃段，基于HashMap原理的并发实现

      加锁只针对table[]中对应的Entry进行加锁，新来的元素放到链表的后面。

  * hsah(key)，计算hash值，通过高位，找到对应的segment[]位置，然后再通过低位找到table[]位置。

  * CHM是弱一致性的

    * 添加元素后不一定马上能读到
    * 清空后可能仍然会有元素
    * 遍历之前的段元素的变化可以读到
    * 遍历之后的段元素的变化不可以读到
    * 遍历时元素发生变化不抛出异常

  * HashTable的问题

    * 大锁：对HashTable对象加锁，虽然线程安全，但是想操作它，都需要拿锁，存在一定的浪费。

* **4.AtomicReference和AtomicReferenceFieldUpdater有何区别**

  * AR和ARFU的功能一致，原理相同，都基于Unsafe的CAS操作
  * AR通常做为对象的成员使用，占16B（指针压缩）、24B（指针不压缩）
  * ARFU通常做为类的静态成员使用，对实例成员进行修改
  * AR使用更友好，ARFU更适合类实例比较多的场景。

* **5.如何在Android中写出优雅的异步代码？**

  * 什么是异步：一个异步过程调用出发后，调用者还没有得到结果之前，就可以继续执行后续操作，如回调。取决于是不是按照顺序执行。
  * 异步的目的：提升CPU利用率；提升GUI程序的响应速度；异步不一定快。
  * RxJava解决回调地狱的问题，链式调用。

## 第五章 JNI编程的细节

* **1.CPU架构适配需要注意什么？**

* **2.Java Native方法与Native函数如何绑定？**

  * 静态绑定：通过命名规则映射

    ```java
    package com.nativec.test;
    public class JniTest {
        public static native void callNativeStatic();
    }
    //**************************** JNI *****************************
    extern "C" JNIEXPORT void JNICALL
    java_com_nativec_test_JniTest_callNativeStatic(JNIENV *, jclass)
    // jclass是实例引用，非静态方法
    // jobject是类引用，静态方法。
    // extern "C"是为了告诉编译期，按照"C"的方式去编译，保留命名规则，C++混编，会找不到路径。
    // JNIEXPORT 为了保证在符号表可见，便于查找（符号表过多，会占用一定的so库体积）
    ```

  * 动态绑定：通过JNI函数注册 

    * 可以在任何时候触发，so库可以被动态替换掉，动态修改native函数调用的目标，
    * 动态绑定之前根据静态规则查找Native函数。
    * 动态绑定可以在绑定后的任意时刻取消。取消后变成静态查找的规则。

  * 静态绑定和动态绑定的对比

    |                    |     动态绑定     |               静态绑定                |
    | ------------------ | :--------------: | :-----------------------------------: |
    | Native函数名       |      无要求      | 按照规定规则编写且采用C的名称修饰规则 |
    | Native函数可见性   |      无要求      |                 可见                  |
    | 动态更换           |       可以       |                不可以                 |
    | 调用性能           |     无需查找     |           有额外的查找开销            |
    | 开发体验           |   几乎无副作用   |          重构代码时较为繁琐           |
    | Android Studio支持 | 不能自动关联跳转 |        自动关联JNI函数，可跳转        |

* **3.JNI如何实现数据的传递？**

  * 

* **4.**

## 第六章 Activity

* **1.Activity的启动流程是怎么样的**？

  * 是否熟悉Activity启动过程中与AMS的交互过程

  * 是否熟悉Binder通信机制

    * 大小受缓冲区大小限制
    * 数据必须可以序列化

  * Activity实例化

    ```java
    return (Activity)cl.loadClass(className).newInstance();
    ```

    `newInstance()`能调用是因为有一个默认的无参构造函数。

    * **Q**：Fragment为什么不能添加有参数的构造函数？

      **A**：Activity被意外杀死，状态保存时，将现有的fragment保存到一个`android:fragment`为key的数据中，包括有哪些fragment，它们的顺序以及状态。当Activity恢复的时候，再把这些fragment重新new出来，但是参数却不知道怎么来。所以最好不要添加有参数的构造函数。

  * 是否了解插件化框架如何Hook Activity启动

  * 阐述Activity转场动画的实现原理

  * 阐述Activity的窗口显示流程可加分

* **2.如何跨App启动Activity**？

  * 是否了解如何启动外部应用的Activity？

    * 共享uid的App

      清单文件根节点添加`android:sharedUserId="**.**"`

    * 使用exported

      Activity添加`android:exported="true"`，使整个Activity暴露在系统中。如微信登录

    * 使用intentFilter

      通过设置`action`来启动

  * 为允许外部启动的Activity加权限

    ```java
    <permission andorid:name="com.start.b"/>
    <activity android:name="BActivity" android:permission="com.start.b">
    ```

    AppB中的BActivity添加了启动权限。

    AppA中想要启动BActivity的时候需要声明权限

    ```java
    <uses-permission android:name="com.start.b"/>
    ```

  * 是否了解如何防止自己的Activity被外部非正当启动

  * 是否对拒绝服务漏洞有了解

    > 假设App A启动App B的时候，在Bundle中传递了一个`SerializableA`的数据类，但是该类只在App A中存在，当传递到App B中，进行访问，反序列化的时候，就会报找不到类的异常，这就是所谓的**拒绝服务**漏洞。

  * 如何在开发时避免拒绝服务漏洞

    * try/catch

  * 尽量不要暴露Activity，如果必须，加上权限控制

* **3.如何解决Activity参数类型安全及接口繁琐的问题**？

  * **类型安全**：Bundle的K-V不能在编译期保证类型

  * **接口繁琐**：启动Activity时参数和结果传递都依赖Intent

  * 为什么Activity的参数存在类型安全的问题？

    `putExtra()`和`getExtra()`，需要人工保证参数类型一致。

    ```java
    // 常规的启动写法
    public class UserActivity extends Activity {
        String name; // 必选
        int age; // 必选
        String title; // 可选 
        String compang; // 可选
        ...
    }
    
    Intent intent = new Intent(context, UserActivity.class);
    intent.putExtra("age", age);
    intent.putExtra("name", name);
    intent.putExtra("title", title);
    context.startActivity(intent);
    ```

    ```java
    // 期望的启动写法
    UserActivityBuilder.builder(age, name)
        .title(title)
        .company(company)
        .start(context);	
    ```

    [参考：ActivityStarter](https://github.com/MarcinMoskala/ActivityStarter)

    元编程

* **4.如何在代码的任意位置为当前的Activity添加View**？

  * 如何获取当前Activity

    `ActivityLifecycleCallbacks`生命周期中获取当前Activity。为了避免Activity泄露，使用弱引用。

  * 如何在不影响正常View展示的情况下添加View

    * 通过`findViewById(android.R.id.content)`拿到父View

    * 通过`addView`和`removeView`来添加或移除View

  * 目的是什么？添加全局View是否更合适？

* **5.如何实现类似微信右滑返回的效果**

  * 没有明说UI的类型，Activity还是Fragment？

  * Fragment实现简单，重点回答Activity的实现

    * Fragment的实现

      * 不涉及Widnow的控制，只是View级别的操作
      * 实现View跟随手势滑动移动的效果
      * 实现手势结束后判断取消或返回执行归位动画

    * Activity的实现

      * ```java
        <item name="android:windowBackground">@android:color/transparent</item>
        <item name="android:windowIsTranslucent">true</item> 
        ```

        当前画布不透明，前一个Activity没绘制

      * Activity联动——多Task

        假设，A、C在Task#0中，B在Task#1中，过程时A跳转到B，B在跳转到C。当从B跳转到C时，会先从Task#1切换到Task#0，此时Task#0中已经存在了A页面，所以，跳转C页面之前，会先出现页面A。

        解决上述问题，需要在跳转动画的时候，先将B页面拍一张照片，放到C的下面，造成一种假想。

        在回调生命周期的时候，通过`getTaskId()`方法获取到该Activity在那个任务栈中。

  * 考虑如何设计成一个组件

    SwipeBackLayout

    * 滑动的时候，将页面设置为透明，不滑动的时候，设置为不透明，避免在不滑动的时候，多余的绘制下层Activity

  * 考虑如何降低接入成本

## 第七章 Handler

* **1.Android非UI线程为什么不能更新UI？**

  * UI线程是什么：zygote通过fork出新的进程，然后新的进程所运行的ActivityThread线程，就是主线程。main方法运行的线程。
  * UI为什么不设计成线程安全？
    * UI具有可变性，甚至是高频可变性
    * UI对响应时间的敏感性要求UI操作必须高效
    * UI组件必须批量绘制来保证效率
  * 非UI线程一定不能更新UI吗？
    * Handler.post / sendMessage
    * View.postInvalidate
    * Activity.runOnUiThread
    * SurfaceView

* **2.Handler发送消息的delay靠谱吗？**

  * 消息空闲：IdleHandler

  * 使用独享的Looper

    ```java
    // HandlerThread构造函数的参数，是线程名字，可用于log追踪。
    private HandlerThread handlerThread = new HandlerThread("MapRender-Thread");
    {handlerThread.start();}
    private Handler mapRenderHandler = new Handler(handlerThread.getLooper());
    ```

  * 大于Handler Looper的周期的时候基本可靠（例如主线程 > 50ms）

  * Looper负载越高，任务越容易积压，进而导致卡顿。

  * 不要用handler的delay做为计时的依据。

* **3.主线程的Looper为什么不会导致应用ANR?**

  * ANR是怎么产生的？
    * ANR类型
      * ServiceTimeout
        * 前台服务 20s
        * 后台服务 200s
      * BroadcaseQueue Timeout
        * 前台广播 10s
        * 后台广播 60s
      * ContentProvider Timeout : 10s
      * InputDispatching Timeout : 5s
    * 
  * Looper的工作机制是什么？
  * Looper不会导致应用ANR的本质原因是什么？
  * Looper死循环为什么不会导致CPU占用率高？
    * epoll_wait

* **4.如何简单的实现一个Handler - Looper框架？**

  * Handler的核心能力：
    * 线程间通信
    * 延迟任务执行
  * Looper的核心能力：
  * MessageQueue的核心能力：
    * 持有消息（单链表）
    * 消息按时间排序（优先级）
    * 队列为空时阻塞读取
    * 头结点有延时可以定时阻塞。（DelayQueue）

## 第八章 内存管理

* **1.如何避免OOM？**

  * OOM的产生：
    * 已使用内存 + 新申请内存 > 可分配内存
    * OOM几乎覆盖所有的内存区域，通常指堆内存。
    * Native Heap在物理内存不够时，也会抛出OOM。
  * 避免使用枚举，Enum占用24Bytes，int占用4Bytes。int有类型安全问题，可用@IntDef限制输入。
  * Bitmap的使用
    * 尽量根据实际需求选择合适的分辨率。
    * 注意原始文件分辨率与内存缩放的结果。
    * 不用帧动画，使用代码实现动效
    * 考虑对Bitmap的重采样和复用配置
  * 谨慎的使用多进程（多进程即使只运行很少的代码，但是也会有一些系统的预加载资源。）
  * 谨慎的使用Large Heap 
    * Java虚拟机：-Xmx4096m
    * Android 虚拟机：android:largeHeap="true"
  * 内存优化5R法则
    * Reduce 缩减：降低图片分辨率 / 重采样 / 抽稀策略
    * Reuse 复用：池化策略 / 避免频繁创建对象，减小GC压力。
    * Reycle 回收：主动销毁、结束，避免内存泄露 / 生命周期闭环
    * Refactor 重构：更合适的数据结构 / 更合理的程序架构
    * Revalue 重审：谨慎使用Large Heap / 多进程 / 第三方框架

* **2.如何对图片进行缓存？**

  * LRU（Least Recently Used）
  * LFU（Least Frequently Used）

* **3.如何计算图片占用内存的大小？**

  * 运行时获取Bitmap大小的方法

    ```java
    // 图片占用大小的理论内存
    public final int getByteCount() {
    		if (mRecycled) {
        		return 0;
        }
        // int result permits bitmaps up to 46,340 x 46,340
        return getRowBytes() * getHeight();
    }
    // 图片实际占用多大内存
    public final int getAllocationByteCount() {
    		if (mRecycled) {
        		return 0;
        }
        return nativeGetAllocationByteCount(mNativePtr);
    }
    ```

    * png图片显示格式为ARGB_8888的话，图片占用的内存大小为 h * w * 4；每个像素需要4个字节（8888，就是4个8bit，也就是4个字节）。
    * jpg图片没有透明通道，不能做透底图，png图片有透明通道，即使是jpg的图片，不指定格式的话，默认也是ARGB_8888，但是jpg没有透明通道，A就没有用，只会浪费内存，所以jpg应该使用RGB_565，图片占用内存大小为 h * w * 2，每个像素需要2个字节（565想加为16，也就是两个字节）。
    * 如果格式是png，格式使用RGB_565的话，那显示出来的图片是没有透明通道的。

  * Drawable的图片

    * drawable的density是1；drawable-nodpi是禁止缩放。

    ```java
    // BitmapFactory.java
    public static Bitmap decodeResourceStream(...) {
    		if (opts.inDensity == 0 && value != null) {
    				final int density = value.density;
            if (density == TypedValue.DENSITY_DEFAULT) {
                opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
            } else if (density != TypedValue.DENSITY_NONE) {
                opts.inDensity = density;
            }
        }
        if (opts.inTargetDensity == 0 && res != null) {
            opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
        } 
        return decodeStream(is, pad, opts);
    }
    ```

  * 图片内存体积优化

    * 跟文件存储格式无关
    * 使用inSampleSize 采样：大图 - > 小图
    * 使用矩阵变换来放大图片 ：小图 -> 大图
    * 使用RGB_565来加载不透明图片
    * 使用9-patch图片做背景
    * 不使用图片
      * 优先使用VectorDrawable
      * 时间和技术允许的前提下使用代码编写动画。

  * 索引模式（Indexed Color）

    * 不能放入drawable类似需要缩放的目录中
    * 得到的Bitmap不能用于创建Canvas
    * 从Android 8.1（API 27）开始移出底层Indexed Color

### 第九章 热修复与插件

* **1.如何规避Android P对访问私有API的限制？**

  ```java
  /*
  * @hide
  */
  @SystemApi
  public void convertFromTranslucent() {
      try {
          mTranslucentCallback = null;
          if (ActivityManager.getService().convertFromTranslucent(mToken)) {
              WindowManagerGlobal.getInstance().changeCanvasOpacity(mToken, true);
          }
      } catch (RemoteException e) {
      }
  }
  ```

  @hide注释，private修饰，都是私有API

  * 自行编译系统源码，并导入项目工程（对public hide方法有效）

  * 使用反射访问

    ```java
    Method.setAccessible(true); // 不仅可以绕过访问限制，还可以修改final变量
    ```

  | 白名单   | SDK，所有APP均能访问                                         |
  | :------- | :----------------------------------------------------------- |
  | 浅灰名单 | 仍可以访问的非SDK 函数 / 字段                                |
  | 深灰名单 | 对于目标SDK低于API级别28的应用，允许使用深灰名单接口；<br />对于目标SDK为API28或更高级别的应用，行为与黑名单相同。 |
  | 黑名单   | 受限，无论目标SDK如何，平台将表现为似乎接口并不存在。<br />使用此类成员，都会出发NoSuchMethodError / NoSuchFieldException；<br />获取此类成员对应的class方法和属性列表，亦不包含在内。 |

  * 开源框架 FreeReflection原理剖析
  * GitHub：tiann / FreeReflection
    * 《一种绕过Android P对非SDK接口限制的简单方法》田维术 2018.6.7

* 以Freereflection为例详细分析如何修改Runtime成员

  * 修改hidden_api_policy_ 绕过第一个条件
  * 修改hidden_api_exemptions_ 绕过第三个条件
  * 探讨第二个条件ClassLoader的实现方式
    * Java层直接反射将调用者Class的ClassLoader置为null
    * Native层利用C++对象内存布局直接修改调用者Class的内存地址。

* **2.如何实现换肤功能？**

  * 需要解决的问题？

    * 主题切换
    * 资源加载
    * 热加载还是冷加载
    * 支持哪些类型的资源（Values下的string，Layout下的布局文件，drawable下的图片，style下的主题）
    * 是否支持增量加载。

  * 系统的换肤支持 - Theme

    * 只支持替换主题中配置的属性值
    * 资源中需要主要主动引用这些属性
    * 无法实现主题外部加载、动态下载。

  * 资源加载流程

    ![](https://github.com/NieJianJian/AndroidNotes/blob/master/Picture/resourceloadingprocess.png)

  * 资源缓存替换流

  * Resources包装流

  * AssetManager替换流

  * 方案对比

    |            | 缓存替换流                                                   | Resources包装流                                              | AssetManager替换流                                           |
    | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
    | 工作机制   | 修改Resources的字段                                          | 包装Resources拦截资源加载                                    | AssetPath中添加皮肤包                                        |
    | 刷新方式   | 重绘View                                                     | 重绘View                                                     | 若替换布局，需重启Activity                                   |
    | 方案优势   | 支持图片资源；<br />支持独立打包动态下发。                   | +支持String / Layout                                         | +支持style<br />+支持assets目录下的文件<br />+替换AM实例非常简洁 |
    | 存在问题   | 替换资源受限；<br />Hook过程较为繁琐<br />影响资源加载，入侵性较强 | 资源获取效率有影响；<br />不支持style、assets目录；<br />Resource需要替换多处；<br />Resource包装类代码量大 | 5.0以前不能新增Entry；<br />强依赖编译期资源id的一致性处理   |
    | 资源重定向 | 无此问题                                                     | 运行时动态映射；<br />编译期静态对齐（可选）                 | 编译期静态对齐                                               |

* **3.VirtualApk如何实现插件化？**

  * 插件化框架如何实现插件APK的类加载
  * 插件化框架如何实现插件APK的资源加载
  * 插件化框架如何实现对四大组件的支持。

* **4.Tinker如何实现热修复**

***

##4.高级工程师心法

* 项目优化

  * **1.如何展开优化类的工作？**
    * 
  * **2.**
  * **3.**

* 项目设计

  * **1.如何解答系统设计类的问题？**

    * 项目诞生

      提出想法 -> 可行性研究 -> 需求分析 -> 系统设计 -> 系统开发 -> 迭代维护 -> 系统重审

    * 系统设计步骤

      * 需求：设计（项目需求）一个网络请求框架
      * 流程：关键就是打包请求、建立连接、发送请求、解析结果
      * 细节：请求和响应的数据结构适配能力、请求重试机制、异步处理能力、使用体验优化

    * 如何用Java实现Handler

      * 需求：移植Android Handler到Java平台
      * 流程：关键消息队列、死循环、阻塞和延时
      * 细节：是否需要支持底层、消息队列性能优化、消息实例池化

    * 热修复和插件化

      * 热修复一般都需要，关键看方案选型
        * 业务场景：是否需要立即生效？
        * 是否需要新增或者修改类？
      * 插件化主要考虑体量
        * 是否融合了多条业务线，多团队协作？插件化是个很好的解决方案

    * 脚本化

      * 是否存在大量可模式化的逻辑？
        * 游戏关卡
        * 自定义的UI体系
      * 是否存在大量需要经常调整的策略？
        * 简单的参数调整无法满足

    * 可移植性

    * 性能问题

      google 工程师Jeff Dean 首先在他关于分布式系统的ppt文档列出来的，到处被引用的很多。

      | Operation                                                    | Time  |
      | ------------------------------------------------------------ | ----- |
      | L1 cache reference 读取CPU的一级缓存                         | 0.5ns |
      | Branch mispredict(转移、分支预测)                            | 5ns   |
      | L2 cache reference 读取CPU的二级缓存                         | 7ns   |
      | Mutex lock/unlock 互斥锁\解锁                                | 25ns  |
      | Main memory reference 读取内存数据                           | 100ns |
      | Compress 1K bytes with Zippy 1k字节压缩                      | 3μs   |
      | Send 2K bytes over 1 Gbps network 在1Gbps的网络上发送2k字节  | 20μs  |
      | Read 1 MB sequentially from memory 从内存顺序读取1MB         | 250μs |
      | Round trip within same datacenter 从一个数据中心往返一次，ping一下 | 500μs |
      | Disk seek  磁盘搜索                                          | 10ms  |
      | Read 1 MB sequentially from disk 从磁盘里面读出1MB           | 20ms  |
      | Send packet CA->Netherlands->CA 一个包的一次远程访问         | 150ms |

    * 监控

      * 异常捕获以及状态保存恢复

        * Java层异常捕获
        * Native层异常捕获

        假如屏幕录制的应用，录制过程中应用挂掉，可能会播放不了，因为mp4里面有一个索引的区域，录制过程中，索引区域会放在最后写入，如果录制没有正常结束，索引区域也不会写入。

      * 性能监控

      * 优化指标监控

      * 运营数据监控

    * 三个步骤

      明确需求、打通流程、优化细节

    * 十个方面

      并发网络与安全、脚本热修复插件、性能监控可移植、思考过程是重点。

  * **2.如何设计一个短视频APP**

    * 录制的mp4播放优化

      通过服务器将ftyp+mdat+moov的拼接方式改为 ftyp+moov+mdat

      不同系统和版本下的播放器需要播放的前提条件如下表：

      | 播放器               | 播放器行为      |
      | -------------------- | --------------- |
      | iOS                  | 一个GOP（推测） |
      | Android 6.0以下      | 5秒视频数据     |
      | Android 7.0以上      | 一个GOP         |
      | 基于FFmpeg自研播放器 | 关键帧          |

  * **3.设计一个网络请求的框架**

    * 明确需求边界

      * 单向请求还是双向请求？
      * 需要支持哪些应用层的协议（http？https？websocket？...）
      * 是否需要支持自定义协议扩展
      * 是否需要支持异步能力？
      * 运行在什么平台上？（是否可移植？）

    * 增加拦截器

      * 可添加公共数据，可对返回数据进行统一处理
      * 服务器未开发好，可模拟服务数据
      * 添加日志输出

    * 为Http协议增加缓存机制

    * 重试机制（重试间隔随着失败次数增加而增加）

    * 使用注解配置请求

      很多框架都使用注解简化配置，比如Spring、retrofit

    * 第三方扩展

    * 代码设计模式

      * 协议体构建使用Builder模式（比如Okhttp）
      * 数据的传输与拦截使用责任链模式（比如okhttp）
      * 数据序列化类型支持使用适配器模式 

    * 主要设计高级的语法

      * 注解：主要用于接口配置和解析
      * 泛型：主要用于数据类型的适配
      * 反射：读取注解信息、反序列化类型等等

    * DNS增强

      移动解析（HttpDNS）基于Http协议向腾讯云的DNS服务器发送域名解析请求，替代了基于DNS协议向运营商Local DNS发起解析请求的传统方式，可以避免Local DNS造成的域名劫持和跨网访问问题，解决移动互联网服务中域名解析异常带来的困扰。

    

***未读***

* 第五章
* 第九章：第三节、第四节
* 第十章：第三节、第四节









