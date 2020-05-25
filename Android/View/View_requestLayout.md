## requestLayout、invalidate

以下源码研究基于Android API 28。

***

### 1. requestLayout的调用链

调用View的requestLayout方法，代码如下：

```java
@CallSuper
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;
    if (mParent != null && !mParent.isLayoutRequested()) {
        mParent.requestLayout(); // 1
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```

View的requestLayout用`@CallSuper`注释，该注释的意思是，即使子类重写，也一定要调用super。

View的requestLayout方法中，有一行代码，如注释1，会调用了mParent的requestLayout方法。mParent的赋值在文章[源码——View之mParent何时赋值](https://github.com/NieJianJian/AndroidNotes/blob/master/Android/View/View_mParent.md)中讲过，最终的根View是DecorView，DecorView的mParent是ViewRootImpl。

在上述代码中，会对View进行向上递归遍历，知道调用到ViewRootImpl的requestLayout方法中，代码如下：

```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}
```

然后调用了scheduleTraversals方法：

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

> Choreographer译为"舞蹈指导"，用于接收显示系统的VSync信号，在下一个帧渲染时执行一些操作。Choreographer的postCallback方法用于发起添加回调，这个添加的回调将在下一帧被渲染时执行。

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
        int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
        int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); // 1
        performLayout(lp, mWidth, mHeight); // 2
        performDraw(); // 3
```

之后就是对View树的重新测量、布局以及绘制。

***

### 2. invalidate的调用链

调用View的invalidate方法：

```java
public void invalidate() {
    invalidate(true);
}
public void invalidate(boolean invalidateCache) {
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}
```

之后调用到了View的invalidateInternal方法：

```java
void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    ...
    if (invalidateCache) { // 1
        mPrivateFlags |= PFLAG_INVALIDATED;
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
    }
    
    final AttachInfo ai = mAttachInfo;
    final ViewParent p = mParent;
    if (p != null && ai != null && l < r && t < b) {
        final Rect damage = ai.mTmpInvalRect;
        damage.set(l, t, r, b);
        p.invalidateChild(this, damage); // 2
    }
    ...
}
```

注释1处判断是否非废除掉缓存，调用invalidate方法，之后传递进来的为true，所以会执行判断内的代码，然后打上`PFLAG_INVALIDATED`标签，并清除掉`PFLAG_DRAWING_CACHE_VALID`标签。

注释2处将需要重绘的举行，传递给父View，之后调用ViewGroup的invalidateChild方法：

```java
public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null && attachInfo.mHardwareAccelerated) { // 1 硬件加速器是否开启
        onDescendantInvalidated(child, child); // 2 硬件加速器的快捷执行路径。
        return;
    }
    ViewParent parent = this;
    ...
    do {
        View view = null;
        if (parent instanceof View) { // 3
            view = (View) parent;
        }
        if (drawAnimation) { // 4
            if (view != null) {
                view.mPrivateFlags |= PFLAG_DRAW_ANIMATION;
            } else if (parent instanceof ViewRootImpl) {
                ((ViewRootImpl) parent).mIsAnimating = true;
            }
        }
        if (view != null) { // 5
            if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                    view.getSolidColor() == 0) {
                opaqueFlag = PFLAG_DIRTY;
            }
            if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
            }
        }
        parent = parent.invalidateChildInParent(location, dirty); // 6
        if (view != null) {
            Matrix m = view.getMatrix();
            if (!m.isIdentity()) {
                RectF boundingRect = attachInfo.mTmpTransformRect;
                boundingRect.set(dirty);
                m.mapRect(boundingRect);
                dirty.set((int) Math.floor(boundingRect.left),
                        (int) Math.floor(boundingRect.top),
                        (int) Math.ceil(boundingRect.right),
                        (int) Math.ceil(boundingRect.bottom));
            }
        }
    } while (parent != null);
}
```

在注释1处判断是否开启了硬件加速器的选项，如果是的话，就执行注释2的方法onDescendantInvalidated，然后return。只有在没有开启硬件加速起的情况下，才会继续往下执行。

注释3处判断parent的类型，之前的都为ViewGroup，最后一次将会是ViewRootImpl类型。

注释4处判断是否正在执行动画，如果是的话设置父View的动画执行标识。

注释6处将调用ViewParent的invalidateChildInParent方法，ViewGroup和ViewRootImpl都实现了ViewParent接口，所以它们也都实现了invalidateChildInParent方法，该方法会返回当前View的父View，一直循环向上，直到返回ViewRootImpl实例。来看下ViewRootImpl的invalidateChildInParent方法：

```java
 public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    checkThread();
    ...
    invalidateRectOnScreen(dirty);
    return null;
}
```

然后调用到了invalidateRectOnScreen方法：

```java
private void invalidateRectOnScreen(Rect dirty) {
    ...
    scheduleTraversals();
}
```

到这里看到和requestLayout一样，也是调用到了scheduleTraversals方法，之后执行performTraversals方法。

同样，我们如果开启了硬件加速器，将会执行invalidateChild方法中的注释2的onDescendantInvalidated方法，

```java
public void onDescendantInvalidated(@NonNull View child, @NonNull View target) {
    ...
    if (mParent != null) {
        mParent.onDescendantInvalidated(this, target);
    }
}
```

onDescendantInvalidated方法中也是调用父View的onDescendantInvalidated方法，一层一层的向上调用，直到ViewRootImpl的onDescendantInvalidated方法：

```
public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
    if ((descendant.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
        mIsAnimating = true;
    }
    invalidate();
}

void invalidate() {
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        scheduleTraversals();
    }
}
```

最终也是调用到了scheduleTraversals方法。

***

### 3. performTraversals方法分析

当我们调用View的invalidate方法时，会调用到ViewRootImpl的scheduleTraversals方法；

当我们调用到View的requestLayout方法时，会调用到ViewRootImpl的requestLayout，之后调用scheduleTraversals方法，但是我们注意到，在ViewRootImpl的requestLayout中调用scheduleTraversals方法之前，有如下一行代码：

```java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true; // 1
        scheduleTraversals();
    }
}
```

**注释1处将mLayoutRequested变量设为了true，但是在invalidate调用链中却没有。这个变量将影响我们的performTraversals方法的执行，直接决定performMeasure、performLayout、performDraw三个方法的执行与否。这也是requestLayout和invalidate方法的区别所在**。

接下来我们看performTraversals的方法：

```java
private void performTraversals() {
    ...
    if (layoutRequested) { 
        // 清除该标志，保证在剩余的执行部分中，有其他的要求改变布局的请求，可以正确执行
        mLayoutRequested = false;
    }
    ...
    if (mFirst || windowShouldResize || insetsChanged ||
            viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
        if (!mStopped || mReportNextDraw) {
            boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
                    (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
            if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
                    || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
                    updatedConfiguration) {
                int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
                int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec); // 1
                ...
                layoutRequested = true; // 2
            }
        }
    } 

    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    if (didLayout) {
        performLayout(lp, mWidth, mHeight); // 3
    }
  
    ...
      
    if (!cancelDraw && !newSurface) {
        if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
            for (int i = 0; i < mPendingTransitions.size(); ++i) {
                mPendingTransitions.get(i).startChangingAnimations();
            }
            mPendingTransitions.clear();
        }
        performDraw(); // 4
    }
    ...
}
```

上面是performTraversals方法大概的执行过程，主要展示了measure、layout、draw三大方法是否执行的判断因素。首先我们可以确定的一点是，`!mStopped || mReportNextDraw`表达式为true的时候，才有可能执行performMeasure方法，一旦执行了performMeasure方法，`layoutRequested`变量就置为true，所以判断performLayout是否执行的变量`didLayout`一定是true。基本可以认为，performMeasure和performLayout是一起被执行的，但是也有不执行performMeasure方法，只执行performLayout方法的时候。

接下来分析ViewRootImpl的几个成员变量：

* mFirst：是否第一次执行，在ViewRootImpl的构造方法中初始化为true。在performTraversals方法的最后部分设为了false，也就说，执行过一次performTraversals，这个变量就永久为false了。
* mStopped：默认为false。代表当前窗口是否是停止状态。只有在当前Activity执行onStop的时候，才会将这个值设置为true。
* mReportNextDraw：默认为false，意义不大（PS：主要是我也不是很理解），因为通常判断条件是mReportNextDraw和mStopped同时处理，mStopped一般为false，!mStopped则为true，一般只要判断!mStopped就能这场执行逻辑，所以暂时不用考虑mReportNextDraw变量。
* mForceNextWindowRelayout：默认为false，只有在Configurationg改变才会设为true。

上面分析完成员变量，接着分析判断着用到的几个局部变量：

* windowShouldResize：顾名思义，窗口应该重新计算大小。它的赋值如下：

  ```java
  boolean windowShouldResize = layoutRequested && windowSizeMayChange
      && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
          || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
                  frame.width() < desiredWindowWidth && frame.width() != mWidth)
          || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
                  frame.height() < desiredWindowHeight && frame.height() != mHeight));
  windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;
  windowShouldResize |= mActivityRelaunched;
  ```

  * layoutRequested：其实就算是`mLayoutRequested`的值。
  * windowSizeMayChange：窗口大小可能发生改变。
  * 剩下的就是宽高方面的对比是否发生了改变。

* insetsChanged：默认为false，主要判断相应的Rect实例是否发生改变。

  ```
  if (!mPendingOverscanInsets.equals(mAttachInfo.mOverscanInsets)) {
      insetsChanged = true;
  }
  if (!mPendingContentInsets.equals(mAttachInfo.mContentInsets)) {
      insetsChanged = true;
  }
  if (!mPendingStableInsets.equals(mAttachInfo.mStableInsets)) {
      insetsChanged = true;
  }
  if (!mPendingDisplayCutout.equals(mAttachInfo.mDisplayCutout)) {
      insetsChanged = true;
  }
  if (!mPendingVisibleInsets.equals(mAttachInfo.mVisibleInsets)) {
      mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
      if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
              + mAttachInfo.mVisibleInsets);
  }
  if (!mPendingOutsets.equals(mAttachInfo.mOutsets)) {
      insetsChanged = true;
  }
  if (mPendingAlwaysConsumeNavBar != mAttachInfo.mAlwaysConsumeNavBar) {
      insetsChanged = true;
  }
  ```

* viewVisibilityChanged：默认false，可见度是否发生改变。

接下来看看决定performDraw是否执行的两个局部变量：

```java
if (!cancelDraw && !newSurface) {
```

* cancelDraw：顾名思义，是否取消绘制，来看看决定条件：

  ```java
  boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
  ```

  * 可见度为gone的情况下，将不会执行draw操作。

  * mAttachInfo.mTreeObserver.dispatchOnPreDraw()主要是判断，有没有实现OnPreDrawListener并且返回fasle的。

    ```java
    public final boolean dispatchOnPreDraw() {
        boolean cancelDraw = false;
        final CopyOnWriteArray<OnPreDrawListener> listeners = mOnPreDrawListeners;
        if (listeners != null && listeners.size() > 0) {
            CopyOnWriteArray.Access<OnPreDrawListener> access = listeners.start();
            try {
                int count = access.size();
                for (int i = 0; i < count; i++) {
                    cancelDraw |= !(access.get(i).onPreDraw()); // 1
                }
            } finally {
                listeners.end();
            }
        }
        return cancelDraw;
    }
    ```

    在注释1处调用OnPreDrawListener的onPreDraw方法，当该方法返回false时，就满足了cancelDraw为true的情况。默认情况下没有实现OnPreDrawListener接口，mOnPreDrawListeners将为null，直接返回false。如下实现，将会改变这个值：

    ```java
    ViewTreeObserver observer = mTextView.getViewTreeObserver();
    observer.addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
        @Override
        public boolean onPreDraw() {
            return false;
        }
    });
    ```

    在onPreDraw方法中返回false，将会导致cancelDraw的值为true，也就不会执行performDraw方法了。

* newSurface：我根据源码的追踪分析，这个值一般为false，只有在页面第一次绘制的时候，也就是mFirst的时候，追踪到了它的变化，它的变化路径如下：

  ```java
  ... 
  if (mFirst || windowShouldResize || insetsChanged ||
                  viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
      ...
      boolean hadSurface = mSurface.isValid(); // 1
      ...
      relayoutResult = relayoutWindow(params, viewVisibility, insetsPending); // 2
      ...
      if (!hadSurface) { // 3
          if (mSurface.isValid()) { // 4
              newSurface = true; // 5
              mFullRedrawNeeded = true;
              mPreviousTransparentRegion.setEmpty();
      ...
  ```

  根据第一个if判断我们看到，这个是进入可以调用performMeasure的前提。

  注释1处，调用`mSurface.isValid()`，判断当前的Surface是否可用；如果是mFirst为true，也就是第一次进入的时候，这个值返回的是false。

  注释2的调用，会改变Surface的一个变量，使得下一次`mSurface.isValid()`返回true，我们稍后分析。

  由于注释1和注释2的执行，注释3和注释4的条件都可以满足，所以在注释5处，newSurface会被设置为true，在这种情况下，不会执行performDraw方法。

  我们来分析注释2：

  ```java
  public boolean isValid() {
      synchronized (mLock) {
          if (mNativeObject == 0) return false;
          return nativeIsValid(mNativeObject);
      }
  }
  ```

  Surface的isValid的，首先会判断mNativeObject值是否会等于0，等于0则返回false；

  在注释3的代码，会经过一系列执行，将mNativeObject变量设置为一个不为0的值，所以下一次isValid判断也就是不会返回false了。

***

### 4. postInvalidate方法调用链

调用View的postInvalidate方法：

```java
public void postInvalidate() {
    postInvalidateDelayed(0);
}
```

```java
public void postInvalidateDelayed(long delayMilliseconds) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    }
}
```

```java
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
    Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
    mHandler.sendMessageDelayed(msg, delayMilliseconds);
}
```

调用Handler，将消息分发出去，来看看mHandler：

```java
final ViewRootHandler mHandler = new ViewRootHandler();

final class ViewRootHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_INVALIDATE:
                ((View) msg.obj).invalidate();
                break;
```

我们看到，View作为Message的一个参数，通过Handler发送和接收，然后在主线程调用View的invalidate。postInvalidate和invalidate的区别就是，postInvalidate可以在子线程中使用。

***

### 参考文章

* [invalidate和requestLayout](https://www.cnblogs.com/genggeng/p/10014121.html)
* [invalidate和requestLayout原理与区别总结](https://www.jianshu.com/p/4f0f0b64381d)
* [invalidate、postInvalidate与requestLayout浅析](https://juejin.im/post/5d53ddd6f265da03d15549b8)
* [你需要了解下Android View的更新 requestLayout 与重绘 invalidate](https://mp.weixin.qq.com/s/_A2PPOam9HIfV-_AFLdgiw)
* [view系列疑惑之关于onmeasure,onLayout, requestLayout ,invalidate你可能忽视的细节](https://www.jianshu.com/p/a416812d2cd2)

