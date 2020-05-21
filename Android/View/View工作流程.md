## View工作流程

View的工作流程主要是指measure、layout、draw这三大流程，即测量、布局和绘制。

* measure过程决定了View的宽高，measure完成后，就可以通过getMeasuredWidth和getMeasuredHeight方法来获取到View测量后的宽高。在几乎所有情况下它都等同于View的最终宽高。除非重写View的layout方法，改变其宽高。
* layout过程决定了View的四个定点的坐标和实际的View的宽高，完成以后就可以通过getTop、getBottom、getLeft和getRight来拿到View的四个顶点的位置，并且可以通过getWidth和getHeight方法来拿到View的最终宽高。
* draw过程则决定了View的显示，只有draw方法完成以后View的内容才能呈现在屏幕上。

### 1. measure过程

measure过程需要分两种情况：

* 一种是原始View，通过measure方法就完成其测量过程
* 一种是ViewGroup，除了完成自己的测量过程，还要去遍历子View的measure方法，各个子View再去递归执行这个过程。

#### 1.1 View的measure过程

View的measure过程由其measure方法来完成，measure方法是一个final方法，无法重写，在measure方法中会调用View的onMeasure方法，代码如下：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

onMeasure的作用就是获取View的宽高值，并作为参数设置给setMeasuredDimension方法。

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
    boolean optical = isLayoutModeOptical(this);
    if (optical != isLayoutModeOptical(mParent)) {
        Insets insets = getOpticalInsets();
        int opticalWidth  = insets.left + insets.right;
        int opticalHeight = insets.top  + insets.bottom;
        measuredWidth  += optical ? opticalWidth  : -opticalWidth;
        measuredHeight += optical ? opticalHeight : -opticalHeight;
    }
    setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}
```

setMeasuredDimension方法显然是用来设置宽高的。重点是我们如何获取测量的宽高，来看getDefaultSize方法

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

关于MesaureSpec的相关知识，可以参考文章[源码——理解MeasureSpec](https://github.com/NieJianJian/AndroidNotes/blob/master/Android/View/MeasureSpec.md)的内容，这篇文章中，也讲到了ViewGroup的measureChildWithMargins方法中，获取了View的宽高的MeasureSpec，并且调用了View的measure方法，将宽高的MeasureSpec，随后View同样将宽高的MeasureSpec传递给了onMeasure方法。所以在调用View的onMeasure之前，View的MeasureSpec已经确定了。

根据getDefaultSize方法中的操作我们可以看到，在AT_MOST和EXACTLY模式下，MeasureSpec中的specSize就是我们该方法的返回值，也就是我们之前测量的View的大小，但是View的最终大小是在layout阶段确定的，不过几乎所有情况的View的测量大小和最终大小都是相等的。

UNSPECIFIED一般用于系统内部测量过程，这种情况下，View的宽高为getSuggestedMinimumWidth和getSuggestedMinimumHeight这两个方法的返回值。来分析下getSuggestedMinimumWidth方法：

```java
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? 
            mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}
```

从上述代码可以看出：

* View没有设置背景，View的宽度为mMinWitdh，对应于android:minWidth属性指定的指，如果这个属性不执行，mMinWidth默认为0；

* View设置背景，View的宽度为max(mMinWidth, mBackground.getMinimumWidth())。mMinWidth我们知道了，来看看mBackground.getMinimumWidth()：

  ```java
  public int getMinimumWidth() {
      final int intrinsicWidth = getIntrinsicWidth();
      return intrinsicWidth > 0 ? intrinsicWidth : 0;
  }
  ```

  getMinimumWidth方法返回的是Drawable的固有宽度，前提是有固有宽度，否则为0。ShapeDrawable无固有宽高，而BiemapDrawable有固有宽高。

  这里总结下：**如果View没有设置背景，那么返回的是android:minWidth这个属性的指，没有设置这个值则为0；如果View设置了背景，则返回android:minWidth和背景的固有宽度（固有宽度可以为0）这两者的最大值**。

根据getDefaultSize方法的实现可以得出一个结论：***直接继承View的自定义空间需要重写onMeasure方法并设置wrap_content时的自身大小，否则在布局中使用wrap_content相当于使用match_parent***。所以，重写View的onMeasure方法，是为了能够给View一个wrap_content属性下的默认大小。

#### 1.2 ViewGroup的measure过程

ViewGroup除了完成自己的measure过程，还会遍历去调用子View的measure方法。

ViewGroup没有重写View的onMeasure方法，它是一个抽象类，其测量过程的onMeasure方法需要各个子类去具体实现，比如LinearLayout、RelativeLayout等。因为不同的ViewGroup子类有不同的布局特性，这导致它们的测量细节各不相同，很难统一。

接下来我们看LinearLayout的onMeasure方法的实现：

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    if (mOrientation == VERTICAL) {
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    } else {
        measureHorizontal(widthMeasureSpec, heightMeasureSpec);
    }
}
```

这个方法很简单，根据垂直还是水平来调用相应的方法，我们来看看measureVertical方法：

```java
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    mTotalLength = 0;
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        ...
        final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
        measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                heightMeasureSpec, usedHeight);
        final int childHeight = child.getMeasuredHeight();
        final int totalLength = mTotalLength;
        mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                lp.bottomMargin + getNextLocationOffset(child));
    ...
}
```

上述代码看出，系统会遍历子View并对每个子View执行measureChildBeforLayout，然后确认它们的MeasureSpec并作为参数，调用子View的measure方法，这样每个子View又进入了measure过程。并且系统会通过mTotalLength变量来存储LinearLayout在竖直方向的初步高度，每测量一个子元素，mTotalLength都会增加，增加的部分包括子View的高度以及子View数值方向上的margin等。

当子元素测量完毕后，LinearLayout会根据子View的情况来测量自己的大小，代码如下：

```java
// Add in our padding
mTotalLength += mPaddingTop + mPaddingBottom;
int heightSize = mTotalLength;
// Check against our minimum height
heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
// Reconcile our calculated size with the heightMeasureSpec
int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            heightSizeAndState);
```

* 针对竖直的LinearLayout来说，它的竖屏方向的测量过程遵循View的测量过程。
* 竖直方向上，如果它们的布局高度采用match_parent或者具体数值，那么它的测量过程和View一致，即高度为specSize；
* 如果它的布局采用warp_content，那么它的高度是所有子元素所占高度总和，但是仍不能超过父容器的剩余空间，当然还需要考虑其竖直方向上的padding。

### 2. layout过程

layout方法是用来确定元素的位置。

* View的layout()方法确认自己的位置。
* View和ViewGroup都没有实现onLayout()方法，和onMeasure一样，实现在具体的布局中，如LinearLayout、RelativeLayout等。
* View的layout方法中，确认自身的位置，随后调用onLayout确定子View的位置。

先来看看View的layout方法：

```java
public void layout(int l, int t, int r, int b) {
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
    ...
```

* 首先通过setFrame来确定View自身的四个顶点的位置，View的四个顶点一旦确定，View在父容器中的位置也就确定了。
* 接着调用onLayout方法，用于确定子View的位置，和onMeasure类似。

onLayout在View和ViewGroup中都没有具体实现，我们来看LinearLayout的onLayout方法：

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```

和onMeasure类似，根据方向来调用不同的方法，来看垂直方向的实现：

```java
void layoutVertical(int left, int top, int right, int bottom) {
    ...
    final int count = getVirtualChildCount();
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
            final LinearLayout.LayoutParams lp =
                        (LinearLayout.LayoutParams) child.getLayoutParams();
            ...
            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }
            childTop += lp.topMargin;
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
            i += getChildrenSkipCount(child, i);
        }
    }
}
```

这个方法会遍历子View并调用setChildFrame方法，其中childTop是不断累加的，这样元素才能依次按照垂直方向一个接一个的排列下去而不会重叠。来看setChildFrame方法：

```java
private void setChildFrame(View child, int left, int top, int width, int height) {
    child.layout(left, top, left + width, top + height);
}
```

在setChildFrame方法中，子View调用layout方法来确认自己的位置。该方法中的width和height是测量的宽高，然后会在layout方法中会通过setFrame去设置子元素的四个顶点位置。

```java
mLeft = left;
mTop = top;
mRight = right;
mBottom = bottom;
```

此时View的mLeft、mTop、mRight、mBottom四个变量才赋值。

在我们获取View的宽高的时候，调用getWidth()和getHeight()方法实现如下：

```java
public final int getWidth() {
    return mRight - mLeft;
}
    
public final int getHeight() {
    return mBottom - mTop;
}
```

**所以，只有执行了View的layout方法，才能真正的获取到View的真实宽高**。

经过上面的分析，我们得知：**在View的默认实现中，View的测量宽和最终宽高始终是相等的，只不过测量宽高是在measure过程，最终宽高是在layout过程，即两者的赋值时机不同。所以，日常开发中，我们认为View的测量宽高就等于最终宽高**。但是也有例外：

```java
public void layout(int l, int t, int r, int b) {
    super.layout(l, t, r + 100, b + 100);
}
```

当我们重写了View的layout方法，并手动修改值，会导致测量宽高和最终宽高不一致。

### 3. draw过程

draw方法就是将View绘制到屏幕上面。

官方注释清楚的说明了遵循的几个步骤，如下：

(1) 绘制背景

(2) 保存当前Canvas层

(3) 绘制View的自身内容

(4) 绘制子View

(5) 如果需要，绘制View的褪色边缘，类似于阴影效果

(6) 绘制装饰，如滚动条

接下来我们看draw方法的源码：

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
    /*
     * Draw traversal performs several drawing steps which must be executed
     * in the appropriate order:
     *
     *      1. Draw the background
     *      2. If necessary, save the canvas' layers to prepare for fading
     *      3. Draw view's content
     *      4. Draw children
     *      5. If necessary, draw the fading edges and restore layers
     *      6. Draw decorations (scrollbars for instance)
     */
  
    // Step 1, draw the background, if needed
    int saveCount;
    if (!dirtyOpaque) {
        drawBackground(canvas);
    }

    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        drawAutofilledHighlight(canvas);

        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }

        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);

        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);

        if (debugDraw()) {
            debugDrawFocus(canvas);
        }

        // we're done...
        return;
    }
    ...
}
```

其中第2步和第5步可以跳过，我们不分析，来看下其他的步骤：

* 步骤1：绘制背景

  ```java
  private void drawBackground(Canvas canvas) {
      final Drawable background = mBackground;
      if (background == null) {
          return;
      }
      setBackgroundBounds();
      ...
      final int scrollX = mScrollX;
      final int scrollY = mScrollY;
      if ((scrollX | scrollY) == 0) {
          background.draw(canvas);
      } else {
          canvas.translate(scrollX, scrollY);
          background.draw(canvas);
          canvas.translate(-scrollX, -scrollY);
      }
  }
  ```

  在指定的画布上绘制背景。并且考虑到了偏移参数scrollX和scrollY，如果偏移值不为0，则先将canvas偏移之后再绘制背景。

* 步骤3：绘制View的自身内容

  ```java
  protected void onDraw(Canvas canvas) { }
  ```

  步骤3调用了View的onDraw方法，这个方法是一个空实现，因为不同的View有着不同的实现，所以我们自定义View的时候重写该方法来绘制。

* 步骤4：绘制子View

  ```java
  protected void dispatchDraw(Canvas canvas) { }
  ```

  步骤4调用了dispatchDraw方法来绘制子View，这个方法也是个空实现。ViewGroup实现了该方法，紧接着我们来看ViewGroup的dispatchDraw方法：

  ```java
  protected void dispatchDraw(Canvas canvas) {
      final int childrenCount = mChildrenCount;
      final View[] children = mChildren;
      ...
      for (int i = 0; i < childrenCount; i++) {
          while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
              final View transientChild = mTransientViews.get(transientIndex);
              if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                      transientChild.getAnimation() != null) {
                  more |= drawChild(canvas, transientChild, drawingTime);
              }
  ```

  dispatchDraw方法中对子View进行遍历，然后调用drawChild方法：

  ```java
  protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
      return child.draw(canvas, this, drawingTime);
  }
  ```

  这里调用的是View中三个参数的draw方法，我们来看下：

  ```java
  boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
      ...
      if (!drawingWithDrawingCache) { // 1
          if (drawingWithRenderNode) {
              mPrivateFlags &= ~PFLAG_DIRTY_MASK;
              ((DisplayListCanvas) canvas).drawRenderNode(renderNode);
          } else {
              if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                  mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                  dispatchDraw(canvas);
              } else {
                  draw(canvas);
              }
          }
      } else if (cache != null) {
          mPrivateFlags &= ~PFLAG_DIRTY_MASK;
          if (layerType == LAYER_TYPE_NONE || mLayerPaint == null) {
              Paint cachePaint = parent.mCachePaint;
              if (cachePaint == null) {
                  cachePaint = new Paint();
                  cachePaint.setDither(false);
                  parent.mCachePaint = cachePaint;
              }
              cachePaint.setAlpha((int) (alpha * 255));
              canvas.drawBitmap(cache, 0.0f, 0.0f, cachePaint);
          } else {
              int layerPaintAlpha = mLayerPaint.getAlpha();
              if (alpha < 1) {
                  mLayerPaint.setAlpha((int) (alpha * layerPaintAlpha));
              }
              canvas.drawBitmap(cache, 0.0f, 0.0f, mLayerPaint);
              if (alpha < 1) {
                  mLayerPaint.setAlpha(layerPaintAlpha);
              }
          }
      }
  }
  ```

  该方法，主要内容就是在注释1处，判断有没有缓存，如果没有，就调用draw方法执行正常的绘制流程，如果有缓存就利用缓存显示。

* 步骤6：绘制装饰

  ```java
  public void onDrawForeground(Canvas canvas) {
      onDrawScrollIndicators(canvas);
      onDrawScrollBars(canvas);
      final Drawable foreground=mForegroundInfo != null?mForegroundInfo.mDrawable : null;
      if (foreground != null) {
          if (mForegroundInfo.mBoundsChanged) {
              mForegroundInfo.mBoundsChanged = false;
              final Rect selfBounds = mForegroundInfo.mSelfBounds;
              final Rect overlayBounds = mForegroundInfo.mOverlayBounds;
              if (mForegroundInfo.mInsidePadding) {
                  selfBounds.set(0, 0, getWidth(), getHeight());
              } else {
                  selfBounds.set(getPaddingLeft(), getPaddingTop(),
                          getWidth() - getPaddingRight(), 
                          getHeight() - getPaddingBottom());
              }
              final int ld = getLayoutDirection();
              Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
                      foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
              foreground.setBounds(overlayBounds);
          }
          foreground.draw(canvas);
      }
  }
  ```

  很明显这个方法用于绘制ScrollBar以及其他装饰，并将它们会知在视图内容的上层。

View有一个特殊的方法setWillNotDraw，来看下源码：

```java
/**
 * If this view doesn't do any drawing on its own, set this flag to
 * allow further optimizations. By default, this flag is not set on
 * View, but could be set on some View subclasses such as ViewGroup.
 *
 * Typically, if you override {@link #onDraw(android.graphics.Canvas)}
 * you should clear this flag.
 *
 * @param willNotDraw whether or not this View draw on its own
 */
public void setWillNotDraw(boolean willNotDraw) {
    setFlags(willNotDraw ? WILL_NOT_DRAW : 0, DRAW_MASK);
}
```

从注释中看出，如果一个View不需要绘制任何内容，就可以将调用这个方法，将这个标识为设置为true，系统就会进行相应的优化。

往往设置了WILL_NOT_DRAW之后的优化，都会将View中有如下的判断：

```java
if ((mViewFlags & WILL_NOT_DRAW) != 0 && (mDefaultFocusHighlight == null)
                && (mForegroundInfo == null || mForegroundInfo.mDrawable == null)) {
    mPrivateFlags |= PFLAG_SKIP_DRAW;
}
```

根据上述代码可以看出，如果View设置了WILL_NOT_DRAW，以及经过其他一些属性判断后，就会对

mPrivateFlags变量添加PFLAG_SKIP_DRAW标志，在接下来的View绘制过程中，在必要的地方对PFLAG_SKIP_DRAW标志进行判断，如果包含PFLAG_SKIP_DRAW标志，就会进行相应的优化。

* View中WILL_NOT_DRAW默认为false，也就是没有设置这个标志。

* ViewGroup默认是开启的，在ViewGroup的构造方法中进行设置

  ```
  public ViewGroup(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
      super(context, attrs, defStyleAttr, defStyleRes);
      initViewGroup();
      initFromAttributes(context, attrs, defStyleAttr, defStyleRes);
  }
  private void initViewGroup() {
      // ViewGroup doesn't draw by default
      if (!debugDraw()) {
          setFlags(WILL_NOT_DRAW, DRAW_MASK);
      }
      ...
  ```

所以，我们在自定义ViewGroup中，想要重写draw方法，就需要手动关闭该标志，调用如下代码实现：

```java
setWillNotDraw(false)；
```

