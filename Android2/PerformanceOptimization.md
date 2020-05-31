## Android 性能优化

* 布局优化
* 绘制优化
* 避免Overdraw
* Bitmap优化
* ListView优化
* 内存优化
* APP启动优化——视觉优化（白屏优化）
* APP启动优化——代码优化
* 相关工具

***

### 布局优化

1. 优化布局层级，防止View数高度太高，不宜超过10层。
2. 如果布局中使用LinearLayout和RelativeLayout都可以，那么采用LinearLayout，因为RelativeLayout布局过程耗时更多，onMeasure中会对子View进行两次measure，横向竖向各一次。
3. LinearLayout加了`layout_weight`属性，会导致LinearLayout调用两次measure。
4. Frame'Layout和LinearLayout是一样高效的ViewGroup。
5. 如果单纯的LinearLayout无法满足，需要嵌套方式来完成，建议使用RelativeLayout，减少布局的嵌套。
6. 可复用的组件抽取出来并通过< include>标签使用。
7. 使用< merge>标签减少布局的嵌套层级。
8. 使用ViewStub标签来实现View的延迟加载。

***

### 绘制优化

绘制优化是指View的onDraw方法要避免执行大量的操作。

1. onDraw尽量不要创建局部变量，因为onDraw方法可能被频繁调用，这样就会产生大量的临时对象，这样不仅占用很多内存，还会导致系统频繁gc。
2. onDraw不要做耗时任务，也不要执行过多的循环操作，否则会导致绘制不流畅（所谓的卡顿）。

Google官方给出的性能优化典范中的标准，View的绘制帧率保证60fps是最佳的（视觉上的流畅画面，需要帧数达到40fps到60fps），这就要求每帧的绘制时间不要超过16ms（16 = 1000 / 60），虽然很难保证，但是尽量降低onDraw方法的复杂度。

**画面卡顿的原因**：Android中，系统通过VSYNC信号触发对UI渲染、重绘，其时间间隔就是16ms。如果系统每次渲染的时间间隔都保持在16ms内，我们看到的UI界面将是非常流畅的。如果16ms无法完成绘制，那么就会导致丢帧现象，即当前该重绘的帧被未完成的逻辑阻塞，例如一次绘制任务耗时20ms，那么在16ms的系统发出VSYNC信号是就因为阻塞而无法绘制，该帧就会被丢弃，等待下次信号才开始重绘，导致2*16ms内显示同一帧画面，这就是**画面卡顿的原因**。

***

### 避免Overdraw

**什么是过度绘制**：屏幕上某一个像素点在同一帧的时间被多次绘制。在多层次重叠的UI结构里面，如果不可见的UI也在做绘制的操作，会导致某些像素区域被绘制了多次。

* 当布局中有多重背景时会导致视图的过度绘制，通过删除删除布局中不需要的背景来减少视图的过度绘制。
* 在布局中，如果存在多个线性布局重叠时，可以考虑只针对最上层的布局设置背景色，而不需要每一个布局（例如LinearLayout）都设置背景色，过多的相同的背景色会导致过度绘制。
* 系统默认会绘制Activity的背景，不要再给Activity绘制重叠的背景。
* Android系统会通过避免绘制那些完全不可见的组件来尽量减少消耗，但是自定义的View重写了onDraw方法，则系统无法监测，这时候我们可以通过canvas.clipRect()方法来你的视图定义可绘制的区域的边界，超出的部分会被忽略。

***

### Bitmap优化

* 使用适合当前屏幕分辨率和大小的图片
* 原图高于设备分辨率，或者需要显示的大小小于原图，要进行缩小动作，也就是二尺裁剪。
* 对图像要求不高的地方，尽量降低图片的精度。
* 使用第三方框架，或者三级缓存，可以更好的使用Bitmap。

***

### ListView优化

* 采用ViewHolder并避免在getView方法中执行耗时操作。
* 根据列表的滑动状态来控制异步任务的执行频率，比如滑动过程中不要开启加载图片的任务。
* 开启硬件加速器。

***

### 内存优化

内存优化，其实也可以认为是从代码编写层面入手，进行优化。

Android应用采用沙箱机制，每个应用（进程）分配的内存大小是有限的，当内存太低时就会触发LMK——Low Memory Killer机制。

* 避免[内存泄露](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/MemoryLeak.md)
* 使用的Service是否可以用IntentService代替，因为系统会倾向于保留这个Service所在的线程，如果停止Service失败，将会导致内存泄露。IntentService会自动停止。
* 当UI不再可见时，释放UI上所占用的所有资源。仅仅是所有UI组件都被隐藏（比如home键回到桌面）时接收到onTrimMemory()回调并且参数为`TRIM_MEMORY_UI_HIDDEN`。
* 当内存紧张时，根据onTrimMemory()回调方法中的内存级别来决定释放哪些资源。
* 使用优化的数据容器类，比如SparseArray、SparseBooleanArray等来代替HashMap。通常HashMap更消耗内存，因为它需要一个额外的实例对象来记录Mapping操作。另外，SparseArray更加高效在于它们避免了对key和value的autobox自动装箱，并且避免了装箱后的解箱。
* 避免使用Enum，Enum的内存消耗是普通static常量的2倍。
* 任何Java都会使用大概500字节的内存空间；每个类的实例大约消耗12~16字节；往HashMap添加一个entry需要一个额外占用32字节的entry对象。
* 对常量使用static修饰符。
* 使用静态方法，可以比普通方法提高15%的方法速度。
* 使用ProGuard来剔除不需要的代码。
* 减少不必要的成员变量，如果一个变量可以定义为局部变量，就不要定义为成员变量。
* 对Cursor、Receiver、Sensor、File等对象的使用，要注意它们的创建、回收与注册、解注册。
* 使用SurfaceView来替代View进行大量、频繁的绘图操作。
* 多个线程任务采用线程池，避免创建大量的Thread对象。
* 根据情况适当使用软引用和弱引用。

***

### APP启动优化——视觉优化（白屏优化）

**Q**：为什么出现白屏（或黑屏）？

　　一个应用程序从桌面点击图标开始，经历了：Zygote进程分配新进程 —> Application的构造 —> attachBaseContext()  —> onCreate() —> 第一个Activity的构造  —> onCreate() —> 配置主题中背景等属性 —> onStart() —> onResume() —> 测量布局绘制显示到屏幕上。

　　当上述流程都走完，最终才能看到SplashActivity的第一眼。在构建这一切的时候，屏幕上显示的是手机默认主题颜色——白色或黑色。**这就是为什么APP启动会出现白屏**。

白屏的持续的时间，主要是这一系列过程的耗时，在这期间，如果增加Application和Activity的onCreate的相关方法的执行复杂度，耗时就会增加，相应的白屏的时间也会增加。

实际上，白屏的显示其实是一个StartingWindow，它是一个准备过程，当用户点击应用图标，系统马上显示的就是StartingWindow，当UI的第一帧渲染好之后，StartingWindow会自动移除。

**解决白屏的方案**

首先，Application的默认主题内容如下：

```java
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <!-- Customize your theme here. -->
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
    <item name="colorAccent">@color/colorAccent</item>
</style>
```

```java
<application
    ...
    android:theme="@style/AppTheme">
</application>
```

上述内容也是默认白屏的原因。

* 解决方案一：**设置透明背景主题**

  ```java
  <style name="WelcomeTheme" parent="AppTheme">
      <item name="android:windowFullscreen">true</item> // 冲满屏幕
      <item name="android:windowIsTranslucent">true</item> // 设置背景透明
  </style>
  ```

  ```java
  <application
      ...
      android:theme="@style/WelcomeTheme">
  </application>
  ```

  将App的背景设置为透明的，这样在应用启动的时候，就不会出现白屏现象。

  不过，这样处理的话，虽然不会出现白屏，并不能真正解决问题，启动它仍然需要花费白屏通的时间，点击桌面图标，反而会给人一种停顿的感觉。

* 解决方案二：**设置背景图片**

  这是一种伪解决方案，我们注意下天猫或京东这些应用就是这么处理的。

  ```java
  <style name="WelcomeTheme" parent="AppTheme">
      <item name="android:windowFullscreen">true</item> // 冲满屏幕
      <item name="android:windowBackground">@mipmap/bg_splash</item> // 设置背景透明
  </style>
  ```

  将主题中的背景图片设置成和启动页一样的背景图，这样会加长背景图显示的时长，给人一种直接进入启动页的感觉。

***

### APP启动优化——代码优化

* Android冷启动耗时

  首先我们需要统计冷启动所需要的时间。

  使用adb命令`adb shell am start -S -W 包名/启动类的全限定名`。`-S`表示重启当前应用。

  ```
  mbp:~ nj$ adb shell am start -S -W com.nj.testappli/com.nj.testappli.MainActivity
  Stopping: com.nj.testappli
  Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.nj.testappli/.MainActivity }
  Status: ok
  Activity: com.nj.testappli/.MainActivity
  ThisTime: 556
  TotalTime: 556
  WaitTime: 615
  Complete
  ```

  `ThisTime`代表最后一个Activity的启动耗时；

  `TotalTime`代表一连串的Activity的启动耗时（有几个Activity就统计几个）；

  `ThisTime`和`TotalTime`一般情况下是相等的。只有在MainActivity的onCreate方法中，启动另外一个Activity，并且调用finish方法时，这两个值才会不相等。

  `WaitTime`是`TotalTime`加上应用进程创建过程的耗时。

  `ThisTime`和`TotalTime`的计算在`ActivityRecord.java`的`reportLaunchTimeLocked`方法中：

  ```java
  private void reportLaunchTimeLocked(final long curTime) {
      ...
      final long thisTime = curTime - displayStartTime;
      final long totalTime = entry.mLaunchStartTime != 0
              ? (curTime - entry.mLaunchStartTime) : thisTime;
  ```

  - `curTime`表示该函数调用的时间点.
  - `displayStartTime`表示一连串启动Activity中的最后一个Activity的启动时间点.
  - `mLaunchStartTime`表示一连串启动Activity中第一个Activity的启动时间点

  * 系统日志统计

  如果需要统计从桌面点击图标到Activity启动完毕，可以用`waitTime`，但是系统的启动时间优化不了，所以优化冷启动只需要处理`ThisTime`即可。

* 代码优化

  从Application的attachBaseContext方法、onCreate方法到Activity的onCreate、onStart以及onResume方法的耗时，都计算在了`ThisTime`中，也就是`TotalTime`中。

  我们往往会在onCreate中执行很多初始化操作，比如繁琐的布局初始化、阻塞主线程的UI绘制操作、I/O读写或者网络读写以及其他一些占用主线程的操作。我们可以对初始化操作做一些分类：

  1. 必要的组件一定要在主线程立即初始化，后续要用；

  2. 组件一定要在主线程初始化，但是可以延迟；

     可以使用`handler.postDelayed`来延迟触发初始化，也可以使用IdleHandler在主线程空闲的时候进行初始化。

  3. 组件可以在子线程初始化。

* 启动页优化

  那些只能在主线程立即初始化的操作，我们可以将这些消耗的时间，在其他地方相抵消。

  假设启动页设置为2000ms固定的展示时间，我们可以根据组件初始化的耗时对其进行调整，使其总的时间仍为2000ms。比如组件初始化时间为800ms，那么启动页只需要再展示1200ms即可。

  Application初始化后会调用attachBaseContext方法，我们在该方法中记录下启动时间：

  ```
  @Override
  protected void attachBaseContext(Context base) {
      super.attachBaseContext(base);
  		SPUtil.putLong("application_attach_time", System.currentTimeMillis());
  }
  ```

  在Activity的onWindowFocusChanged方法的回调时机，就是View流程绘制完的时候，所以在这里记录下显示时间：

  ```java
  @Override
  public void onWindowFocusChanged(boolean hasFocus) {
      super.onWindowFocusChanged(hasFocus);
      long appAttachTime = SPUtil.getLong("application_attach_time");
      long diffTime = System.currentTimeMillis() - appAttachTime;
  }
  ```

  所以最后启动页的显示时间为`2000ms - diffTime`的时间。

***

### 相关工具

#### 1. 检测UI渲染时间工具

使用方法：打开手机设置 -> 开发者选择 -> 选择"GPU渲染模式分析" -> 并选中"在屏幕上显示为条形图"。

具体参数分析：[UI优化之-GPU Rendering Profile](https://www.jianshu.com/p/0b90891771e9)

#### 2. 检测Overdraw工具

使用方法：打开手机设置 -> 开发者选择 -> 选择"调试GPU过度绘制" -> 并选中"显示过度绘制区域"。

通过界面上的颜色来判断Overdraw的次数：

* 没颜色：没有过度绘制，也就是一个像素只绘制了一次。
* 蓝色：过度绘制一次，也就是一个像素点绘制了两次。
* 绿色：过度绘制2次，，也就是一个像素点绘制了3次，通常集中优化过度绘制次数大于等于2的情况。
* 浅红色：过度绘制3次。
* 深红色：过度绘制4次，像素点被绘制了5次，甚至更多次。

增大蓝色区域，减少红色区域。

#### 3. Hierarchy Viewer

Android Studio 3.0 以下：Tools -> Android -> Android Device Monitor

Android Studio 3.0 开始：Tools -> Layout Inspector。

* 然后选择相关的应用和进程
* 确定后会在项目目录中生成Captures目录，并且生成相应的后缀为".ii"的快照文件
* 对快找文件进行分析。
* 每次修改布局，都需要重新生成快照。

#### 4. Lint工具

Android Lint 是 SDK Tools 16（ADT 16）开始引入的一个代码扫描工具，通过对代码进行静态分析，可以帮助开发者发现代码质量问题和提出一些改进建议。

* Security 安全性。在AndroidManifest.xml中没有配置相关权限等。
* Usability 易用性。重复图标；上文开始黄色警告也属于该规则等。
* Performance 性能。内存泄漏，xml结构冗余等。
* Correctness 正确性。超版本调用API，设置不正确的属性值等。
* Accessibility 无障碍。单词拼写错误等。
* Internationalization国际化。字符串缺少翻译等。

使用方法：Android Studio菜单栏 -> Analyze -> Inspect Code。

配置Lint文件：

您可以在 `lint.xml` 文件中指定 lint 检查偏好设置。如果您是手动创建此文件，请将其放置在 Android 项目的根目录下。`lint.xml` 文件由封闭的 `< lint>` 父标记组成，此标记包含一个或多个 `< issue>` 子元素。lint 会为每个 `<issue>` 定义唯一的 `id` 属性值。

```xml
<?xml version="1.0" encoding="UTF-8"?>
    <lint>
        <!-- list of issues to configure -->
    </lint>
```

您可以通过在 `<issue>` 标记中设置严重级别属性来更改某个问题的严重级别或对该问题停用 lint 检查。

### 参考链接

* [Android代码内存优化建议-OnTrimMemory优化](https://www.jianshu.com/p/5b30bae0eb49)
* [性能优化工具（六）-Layout Inspector](https://www.jianshu.com/p/1b64024f2d08)
* [通过 lint 检查改进代码](https://developer.android.google.cn/studio/write/lint#studio_config)
* [Android 应用白屏、黑屏、闪屏解决方法 (秒开应用思路)](https://blog.csdn.net/u011418943/article/details/88537446)
* [Android - 启动白屏分析与优化](https://www.jianshu.com/p/cdbdfa5a5319?utm_campaign=haruki&utm_content=note&utm_medium=reader_share&utm_source=qq)
* [Android APP应用启动页白屏(StartingWindow)优化](https://www.cnblogs.com/whycxb/p/9312914.html)
* [Android 性能优化(一) —— 启动优化提升60%](https://blog.csdn.net/qian520ao/article/details/81908505)
* [如何调试Android Framework？](http://weishu.me/2016/05/30/how-to-debug-android-framework/)

