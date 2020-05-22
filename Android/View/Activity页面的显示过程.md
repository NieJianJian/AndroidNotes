## Activity页面的显示过程

### 1. Activity的构成

通常Activity是用setContentView()方法来设置一个布局，我们来看下方法实现，代码如下：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

这里调用了getWindow的setContentView()方法，来看下getWindow：

```java
public Window getWindow() {
    return mWindow;
}
```

mWindow是什么？来看下定义：

```java
private Window mWindow;
```

那在哪里初始化的呢？接着往下看，最后发现在Activity的attach()方法中进行赋值的：

```java
final void attach(...) {
    attachBaseContext(context);
    mFragments.attachHost(null /*parent*/);
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    ...
}
```

说明mWindow，最终指向的是PhoneWindow。Window是一个抽象类，PhoneWindow继承了Window，是Window的具体实现类。最终调用了PhoneWindow的setContentView()方法，继续往下看：

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor(); // 1
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }
    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
    mContentParentExplicitlySet = true;
}
```

接下来看上面注释1处installDecor()方法做了些什么，代码如下：

```java
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor(-1); // 1
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor); // 2
        ...
```

接着看注释1处的generateDecor()方法做了什么：

```java
protected DecorView generateDecor(int featureId) {
    ...
    return new DecorView(context, featureId, this, getAttributes());
}
```

这里创建了一个DecorView，这个DocorView是Activity的根View。

DecorView是Phone的内部类，并且继承了FrameLayout。

接下来回到installDecor()方法中，查看注释2处的generateLayout(mDecor)方法，代码如下：

```java
protected ViewGroup generateLayout(DecorView decor) {
    ...
    int layoutResource;
    int features = getLocalFeatures();
    if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
        layoutResource = R.layout.screen_swipe_dismiss;
        setCloseOnSwipeEnabled(true);
    } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleIconsDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_title_icons;
        }
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
            && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
        layoutResource = R.layout.screen_progress;
    } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogCustomTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else {
            layoutResource = R.layout.screen_custom_title;
        }
        removeFeature(FEATURE_ACTION_BAR);
    } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
        if (mIsFloating) {
            TypedValue res = new TypedValue();
            getContext().getTheme().resolveAttribute(
                    R.attr.dialogTitleDecorLayout, res, true);
            layoutResource = res.resourceId;
        } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
            layoutResource = a.getResourceId(
                    R.styleable.Window_windowActionBarFullscreenDecorLayout,
                    R.layout.screen_action_bar);
        } else {
            layoutResource = R.layout.screen_title; // 1
        }
    } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
        layoutResource = R.layout.screen_simple_overlay_action_mode;
    } else {
        layoutResource = R.layout.screen_simple;
    }
    mDecor.startChanging();
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource); // 2
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT); // 3
    if (contentParent == null) {
        throw new RuntimeException("Window couldn't find content container view");
    }
    ...
    return contentParent;
}
```

该方法的主要作用是根据不同的情况加载不同的布局赋值给layoutResource。然后注释2处调用DecorView的onResourceLoaded方法，该方法内部会调用LayoutInflater.inflate方法，根据传进去的layoutResource生成相应的View；在上述注释3处，会调用findViewById方法，生成contentParent并返回，从而完成了installDecor方法中的mContentParent变量的赋值，我们来看ID_ANDROID_CONTENT是什么：

```java
/**
 * The ID that the main layout in the XML layout file should have.
 */
public static final int ID_ANDROID_CONTENT = com.android.internal.R.id.content;
```

ID_ANDROID_CONTENT是一个id为content的值，现在主要来看注释1处的布局文件R.layout.scrren_title，这个文件在frameworks，代码如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:fitsSystemWindows="true">
    <!-- Popout bar for action modes -->
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
        android:layout_width="match_parent" 
        android:layout_height="?android:attr/windowTitleSize"
        style="?android:attr/windowTitleBackgroundStyle">
        <TextView android:id="@android:id/title" 
            style="?android:attr/windowTitleStyle"
            android:background="@null"
            android:fadingEdge="horizontal"
            android:gravity="center_vertical"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </FrameLayout>
    <FrameLayout android:id="@android:id/content"
        android:layout_width="match_parent" 
        android:layout_height="0dip"
        android:layout_weight="1"
        android:foregroundGravity="fill_horizontal|top"
        android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

上面的ViewStub是用来显示ActionBar的。下面的两个FrameLayout，一个是title，用来显示标题；另一个是content，用来显示内容。

通过上述源码的分析，我们可以知道：

* 每个Activity都包含一个Window对象，这个对象由PhoneWindow来实现；
* PhoneWindow将一个DecorView作为应用窗口的根View；
* DecorView又将屏幕划分为上下两个区域，一个是TitleView，一个ContentView。我们平时通过setContentView()方法来设置的，就是ContentView的内容。

Activity的构成图如下：

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/activitystructure.png" alt="Activity的构成图 " style="zoom:67%;" />

当程序在onCreate()方法中调用setContentView()方法后，ActivityManagerService会回调onResume()方法，此时系统才会把整个DecorView添加到PhoneWindow中，并让其显示出来，从而最终完成界面的绘制。

### 2. DecorView加载到Window中的过程

当调用Activity的startActivity方法时，最终调用的时ActivityThread的handleLaunchActivity方法：

```java
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
    final Activity a = performLaunchActivity(r, customIntent);
    ...
```

调用performLaunchActivity方法来创建Activity，该方法中通过Instrumentation，最终调用了Activity的onCreate方法，从而完成DecorView的创建。

在调用Activity的onResume之前，会先调用ActivityThread的handleResumeActivity方法，代码如下：

```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest,
                                 boolean isForward,String reason) {
    ...
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    if (r == null) {
        return;
    }
    final Activity a = r.activity;
    ...
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView(); // 2
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager(); // 3
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
        l.softInputMode |= forwardBit;
        ...
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l); // 4
            }
    ...
```

performResumeActivity方法最终会调用到Activity的onResume方法。注释2处得到了DecorView。注释3处得到了WindowManager，WindowManager是一个接口，并且继承了ViewManager。注释4处调用WindowManager的addView方法，WindowManager的实现类是WindowManagerImpl，所以调用的是WindowManagerImpl的addView方法。代码如下：

```java
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
    ...
}
```

WindowManagerImpl的addView方法中，又调用了WindowManagerGlobal的addView方法，所以可以看出，WindowManagerImpl虽然是WindowManager的实现类，但是没有实现什么功能，而是将功能委托给了WindowManagerGlobal，这里用到的是桥接方式。上述也可以看出，WindowManagerGlobal是一个单例，说明一个进程只有一个该实例。

通过上述源码分析，WindowManager的关联类如图：

![图：WindowManager的关联类](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/windowmanagerguanlianclass.png)

从上图可以看出：

* PhoneWindow继承自Window；
* Window通过setWindowManager方法与WindowManager发生关联；
* WindowManager继承自接口ViewManager；
* WindowManagerImpl是WindowManager接口的实现类，但是功能委托给WindowManagerGlobal实现。

来看WindowManagerGlobal的addView方法，代码如下：

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    ...
    ViewRootImpl root;
    View panelParentView = null;
    synchronized (mLock) {
        ...
        root = new ViewRootImpl(view.getContext(), display); // 1
        view.setLayoutParams(wparams);
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);
        try {
            root.setView(view, wparams, panelParentView); // 2
        } catch (RuntimeException e) {
            if (index >= 0) {
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

注释1处创建了ViewRootImpl实例，注释2处调用了ViewRootImpl的setView方法并将DecorView作为参数传进去，这样就把DecorView加载到了Window中。当然此时界面仍然什么都不显示，因为View的工作流程还没有完成，还需要经过measure、layout以及draw才会把View绘制出来。

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
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        } 
        if (didLayout) {
            performLayout(lp, mWidth, mHeight);
            ...
        }
        if (!cancelDraw && !newSurface) {
            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).startChangingAnimations();
                }
                mPendingTransitions.clear();
            }

            performDraw();
        }
```

该方法中主要执行了3个方法，分别是performMeasure、performLayout和performDraw，这3个方法完成顶级View的measure、layout和draw这三大流程。其中performMeasure中会调用View的measure方法，在measure方法中会调用onMeasure方法，在onMeasure方法中则会对所有的子元素进行measure过程，这个时候measure流程就从父容器传递到子元素中了，这样就完成了一次measure过程。接着子元素会重复父容器的measure操作，如果反复就完成了整个View树的遍历。performLayout和performDraw的传递流程和performMeasure是类似的，唯一不同的是，performDraw的传递过程是在draw方法通过dispatchDraw来实现的。不过这并没有本质区别。

