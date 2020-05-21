## View的requestLayout方法

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

### 参考文章

* [invalidate和requestLayout](https://www.cnblogs.com/genggeng/p/10014121.html)
* [invalidate和requestLayout原理与区别总结](https://www.jianshu.com/p/4f0f0b64381d)
* [invalidate、postInvalidate与requestLayout浅析](https://juejin.im/post/5d53ddd6f265da03d15549b8)
* [你需要了解下Android View的更新 requestLayout 与重绘 invalidate](https://mp.weixin.qq.com/s/_A2PPOam9HIfV-_AFLdgiw)
* [view系列疑惑之关于onmeasure,onLayout, requestLayout ,invalidate你可能忽视的细节](https://www.jianshu.com/p/a416812d2cd2)

