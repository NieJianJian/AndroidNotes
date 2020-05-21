## View之MeasureSpec

### 1. MeasureSpec介绍

MeasureSpec是一个32位的int值，高2位代表SpecMode（测量的模式），低30位代表SpecSize（测量的大小）。MeasureSpec是View的内部类，来看下源码：

```java
public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;
    public static final int EXACTLY     = 1 << MODE_SHIFT;
    public static final int AT_MOST     = 2 << MODE_SHIFT;
    public static int makeMeasureSpec(int size, int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
}
```

MeasureSpec通过将SpecMode和SpecSize打包成一个int值来避免过多的内存分配，为了方便操作，还提供了打包和解包的方法。

SpecMode有以下三种类型：

* UNSPECIFIED

  父容器不对View大小做任何限制，想多大就多大。一般只会在系统内部，表示一种测量的状态；或者在自定义View的时候，按照自己的意愿设置大小。

* EXACTLY

  父容器已经检测出View所需要的精确大小，这个时候View的最终大小就是SpecSize所指定的值。通常在指定具体数字和match_parent时使用这个模式。

* AT_MOST

  父容器指定一个可用大小，View的大小不能大于这个值，一般将宽高设置为wrap_content时使用。

### 2. MeasureSpec产生过程

我们通过MeasureSpec来对View进行测量，但是MeasureSpec是如何确定的呢？

**View的MeasureSpec需要自身的LayoutParams和父容器的MeasureSpec来决定**。

我们的顶级View（DecorView）和普通View不同，DecorView的MeasureSpec由自身的LayoutParams和窗口的尺寸来共同确定的。

#### 2.1 DecorView的MeasureSpec产生过程

来看ViewRootImpl中的measureHierarchy方法中有如下几行代码：

```java
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

其中`desiredWindowWidth`和`desiredWindowHeight`是屏幕的尺寸，来看下getRootMeasureSpec方法：

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

根据上述代码，我们就可以知道DecorView的产生过程，根据它的LayoutParams中的宽/高来划分

* MATCH_PARENT：精确模式，大小为窗口的大下。
* WRAP_CONTENT：最大模式，大小不定，但是不能超过窗口的大小。
* 固定大小：精确模式，大小为LayoutParams中指定的大小。

根据DecorView中LayoutParams设定的宽高，来确定其Size和Mode，最终调用makeMeasureSpec生成其对应的MeasureSpec。

#### 2.2 View的MeasureSpec的产生过程

View的measure过程由ViewGroup传递而来，先看下ViewGroup的measureChildWithMargins方法：

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

上述方法会对子View进行测量，测量之前，需要先确定子View的MeasureSpec，从代码中可以看出，子View的MeasureSpec的创建与父容器的MeasureSpec和子View本身的LayoutParams有关，此外还和View的margin及padding有关，我们来看ViewGroup的getChildMeasureSpec方法：

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec); // 父容器的大小
    // 父容器去除margin和padding后的实际剩余的大小
    int size = Math.max(0, specSize - padding);
  
    int resultSize = 0;
    int resultMode = 0;
  
    switch (specMode) {
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

上述代码主要就是根据父容器的MeasureSpec结合View自身的LayoutParams来确定View的MeasureSpec，参数中的padding是指父容器已经占用的空间，因此子元素最大可用大小为：

```java
int size = Math.max(0, specSize - padding);
```

下面我们用一个表格来梳理下普通View的MeasureSpec创建规则：

<table>
  	<tr align="center">
      	<th>parentSpecMode</th>
      	<th>childLayoutParams</th>
      	<th>childSpecMode</th>
      	<th>childSpecSize</th>
    </tr>
    <tr align="center">
      	<td  rowspan="3">EXACTLY</td>
      	<td>dp/px</td>
      	<td>EXACTLY</td>
      	<td>childSize</td>
    </tr>
  	<tr align="center">
      	<td>match_parent</td>
      	<td>EXACTLY</td>
      	<td>parentSize</td>
    </tr>
  	<tr align="center">
      	<td>wrap_content</td>
      	<td>AT_MOST</td>
      	<td>parentSize</td>
    </tr>
    <tr align="center">
      	<td  rowspan="3">AT_MOST</td>
      	<td>dp/px</td>
      	<td>EXACTLY</td>
      	<td>childSize</td>
    </tr>
  	<tr align="center">
      	<td>match_parent</td>
      	<td>AT_MOST</td>
      	<td>parentSize</td>
    </tr>
  	<tr align="center">
      	<td>wrap_content</td>
      	<td>AT_MOST</td>
      	<td>parentSize</td>
    </tr>
    <tr align="center">
      	<td  rowspan="3">UNSPECIFIED</td>
      	<td>dp/px</td>
      	<td>EXACTLY</td>
      	<td>childSize</td>
    </tr>
  	<tr align="center">
      	<td>match_parent</td>
      	<td>UNSPECIFIED</td>
      	<td>0</td>
    </tr>
  	<tr align="center">
      	<td>wrap_content</td>
      	<td>UNSPECIFIED</td>
      	<td>0</td>
    </tr>
</table>

childSize是指子View设置的大小，parentSize是指父容器目前可用的大小。

这里来简单总结下：

* 当View采用固定宽/高时，不管父容器的MeasureSpec时什么，View的MeasureSpec都是EXACTLY并且大小遵循LayoutParams中的大小。
* 当View的宽高采用match_parent时，如果父容器的模式是精确模式，那么View也是精确模式，并且其大小是父容器剩余空间；如果父容器是最大模式，那么View也是最大模式并且其大小不会超过父容器剩余空间。
* 当View的宽高采用wrap_content时，不管父容器的模式是精确模式还是最大模式，View的模式总是最大化模式并且大小不能超过父容器的剩余空间。