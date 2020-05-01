## Android 性能优化

* 布局优化
* 绘制优化
* 避免Overdraw
* Bitmap优化
* ListView优化
* 内存优化
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

作者：凯玲之恋
链接：https://www.jianshu.com/p/a0f28fbef73f
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

### 参考链接

* [Android代码内存优化建议-OnTrimMemory优化](https://www.jianshu.com/p/5b30bae0eb49)
* [性能优化工具（六）-Layout Inspector](https://www.jianshu.com/p/1b64024f2d08)
* [通过 lint 检查改进代码](https://developer.android.google.cn/studio/write/lint#studio_config)