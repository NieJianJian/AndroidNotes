## ViewRootImpl

ViewRootImpl的初始化，是在WindowManagerGlobal中。讲ViewRootImpl之前，先讲讲WindowManager。

### 1. WindowManager

创建View，会调用WindowMangaer的addView方法的。WindowManager继承自ViewManager，来看下ViewManager的代码：

```java
public interface ViewManager{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

ViewManager中定义了3个方法的，分别用来添加、更新和删除View。WindowManger继承了这些方法的，而这些方法传入的参数都是View类型，说明**Window是以View的形式存在的**。

Window分为三大类，分别是：

* Application Window（应用程序窗口）

  Activity就是典型的应用程序窗口。

* Sub Window（子窗口）

  子窗口，不能独立存在，需要依附其他窗口，如Popup Window就是是子窗口。

* System Window（系统窗口）

  Toast、输入法窗口、系统音量窗口、系统错误窗口都属于系统窗口。

上述三大窗口的添加都需要经过addView进行添加，之后调用WindowManagerGlobal的addView，进而封装成一个ViewRootImpl对象，调用它的setView方法进行设置。

WindowManagerGlobal中维护的和Window相关的3个列表，在窗口的添加、更新和删除过程中都会涉及

```java
// View列表
private final ArrayList<View> mViews = new ArrayList<View>();
// ViewRootImpl列表
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
// 布局参数列表
private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
```

了解这3个列表后，接着对addView进行分析：

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ...省略参数检查
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    if (parentWindow != null) {
        parentWindow.adjustLayoutParamsForSubWindow(wparams); // 1
    } else {
        final Context context = view.getContext();
        if (context != null&& (context.getApplicationInfo().flags
                        & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
            wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
        }
    }
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        ...
        root = new ViewRootImpl(view.getContext(), display); // 2
        view.setLayoutParams(wparams);
        mViews.add(view); // 3
        mRoots.add(root); // 4
        mParams.add(wparams); // 5
        try {
            root.setView(view, wparams, panelParentView); // 6
        } catch (RuntimeException e) {
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

* 首先对3个参数进行检查
* 注释1处，如果当前窗口要作为子窗口，就会更具父窗口对子窗口的WindowManager.LayoutParams类型的wParams对象进行相应调整。
* 注释3处将添加的View保存到View列表。
* 注释5处将窗口的参数保存到布局参数列表中
* 注释2处将root存入到ViewRootImpl列表中
* 注释6处将窗口的参数通过你过setView方法设置到ViewRootImpl中。

### 2. ViewRootImpl

由上述的接收，可见我们添加窗口这一操作是通过ViewRootImpl来进行的。ViewRootImp的职责主要是：

* View树的根并管理View树
* 触发View的测量、布局和绘制。
* 输入事件的中转站
* 管理Surface
* 负责与WMS进行进程间通信

Activity中，布局是展示在DecorView中的，但实际整个Activity布局的根View，是ViewRootImpl。

同样，即使创建一个系统窗口，窗口显示布局的根View，依旧是ViewRootImpl。

ViewRootImpl作为根View，但并不是View，只是实现了ViewParent接口的类，它的整个绘制，都是通过创建Surface进行绘制的。

我们的WindowManagerGlobal中的ViewRootImpl列表，存放的就是整个进程中的所有窗口View，包括所有Activity页面的DecorView实例，以及子窗口View，以及系统弹窗等。

Window窗口的添加、更新以及删除，最终都会通过ViewRootImpl来进行View操作。

### 3. ViewRootImpl的PerformTraversals方法

将DecorView加载到Window中，是通过ViewRootImpl的setView方法。ViewRootImpl还有一个方法performTraveals，这个方法使得ViewTree开始View的工作流程，

在ViewRootImple的setView方法中，调用了reqeustLayout方法：

```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

随后调用了scheduleTraversals()方法：

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

在scheduleTraversals中，调用了Choreographer的postCallback，该方法最终会调用了主线程的Handler，发送一个Runnable对象去执行run方法，这个Runnable对象就是`mTraversalRunnable`

```java
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

然后调用了doTraversal()方法：

```java
void doTraversal() {
    if (mTraversalScheduled) {
        ...
        performTraversals();
        ...
    }
}
```

最终调用了performTraversals方法，代码如下：

```java
private void performTraversals() {
    ...
        if (!mStopped || mReportNextDraw) {
            ...
            int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
            int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); // 1
        } 
        if (didLayout) {
            performLayout(lp, mWidth, mHeight); // 2
            ...
        }
        if (!cancelDraw && !newSurface) {
            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).startChangingAnimations();
                }
                mPendingTransitions.clear();
            }

            performDraw(); // 3
        }
```

该方法中主要执行了3个方法，分别是performMeasure、performLayout和performDraw，这3个方法完成顶级View的measure、layout和draw这三大流程。其中performMeasure中会调用View的measure方法，在measure方法中会调用onMeasure方法，在onMeasure方法中则会对所有的子元素进行measure过程，这个时候measure流程就从父容器传递到子元素中了，这样就完成了一次measure过程。接着子元素会重复父容器的measure操作，如果反复就完成了整个View树的遍历。performLayout和performDraw的传递流程和performMeasure是类似的，唯一不同的是，performDraw的传递过程是在draw方法通过dispatchDraw来实现的。不过这并没有本质区别。

### 参考文章

* [如果面试官问你：如何理解Window、ViewParent、ViewRootImpl](https://baijiahao.baidu.com/s?id=1652990190210449929&wfr=spider&for=pc)