## View之mParent何时赋值

View中有一个ViewParent类型的变量`mParent`，用来描述当前View的父View：

```java
/**
 * The parent this view is attached to.
 * {@hide}
 *
 * @see #getParent()
 */
protected ViewParent mParent;
```

得到一个View的父View，可以调用View的getParent()方法：

```java
/**
 * Gets the parent of this view. Note that the parent is a
 * ViewParent and not necessarily a View.
 *
 * @return Parent of this view.
 */
public final ViewParent getParent() {
    return mParent;
}
```

那么这个mParent是何时赋值的呢？接下来看View的assignParent方法：

```java
/*
 * Caller is responsible for calling requestLayout if necessary.
 * (This allows addViewInLayout to not request a new layout.)
 */
void assignParent(ViewParent parent) {
    if (mParent == null) {
        mParent = parent;
    } else if (parent == null) {
        mParent = null;
    } else {
        throw new RuntimeException("view " + this + " being added, but"
                + " it already has a parent");
    }
}
```

该方法在源码中的调用位置如下：

* ViewGroup.addViewInner中
* ViewRootImpl.setView中
* ViewRootImpl.dispatchDetachedFromWindow中
* WindowManagerGlobal.removeViewLoacked中

看了上述方法，是不是View的mParent赋值都会调用这个方法？其实不然。接下来需要分析两种情况：

* DecorView的mParent赋值；
* 普通View的mParent赋值。

### 1. DecorView的mParent赋值

首先要根据DecorView的由来分析，在[Activity页面的显示](https://github.com/NieJianJian/AndroidNotes/blob/master/Android/View/Activity页面的显示过程.md)一文中，分析了DecorView的产生过程，我们再来简单回顾一下：

```java
root = new ViewRootImpl(view.getContext(), display); // 1
...
root.setView(view, wparams, panelParentView); // 2
```

在WindowManagerGlobal的addView方法中，创建了一个ViewRootImpl实例，并调用它的setView方法，将view作为参数传进去，此时的view就是DecorView。

在ViewRootImpl的setView方法中，就有如下一行代码：

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
        if (mView == null) {
            mView = view;
            //注意这行代码
            view.assignParent(this);
            ……
        }
    }
}
```

`view`就是DecorView，`this`由于当前代码运行在ViewRootImpl中，由于ViewRootImpl实现了ViewParent接口，所以我们得知，DecorView的`mParent`指向的是ViewRootImpl。

### 2. 普通View的mParent赋值

DecorView是根View，它的mParent指向的是ViewRootImpl，那么DecorView的直接子View的mParent应该指向的就是DecorView，然后一级一级的嵌套下去，那么，具体是怎么实现的呢？

先来分一下以下调用链：

* 在Activity的onCreate方法中，调用setContentView方法：

  ```java
  public void setContentView(@LayoutRes int layoutResID) {
      getWindow().setContentView(layoutResID);
      initWindowDecorActionBar();
  }
  ```

* getWindow我们得知是Window对象，Window的具体实现类是PhoneWindow，来看它的setContentView：

  ```java
  public void setContentView(int layoutResID) {
      if (mContentParent == null) {
          installDecor();
      }
  ```

  里面有一行代码，调用了installDecor()方法

* 我们来看PhoneWindow的installdecor()方法：

  ```java
  private void installDecor() {
      if (mContentParent == null) {
          mContentParent = generateLayout(mDecor);
  ```

  内部调用了generateLayout()方法，并将DecorView传递了进去

* PhoneWindow的generateLayout方法中，有下面一行代码：

  ```java
  mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
  ```

  layoutResource，根据[Activity页面的显示](https://github.com/NieJianJian/AndroidNotes/blob/master/Android/View/Activity页面的显示过程.md)一文中可以看到，是R.layout.scrren_title文件

* 接下来看DecorView的onResourcesLoaded方法：

  ```java
  void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
      final View root = inflater.inflate(layoutResource, null);
  ```

* 之后经过在LayoutInflater中的一系列方法调用，到了rInflate方法

  ```java
  void rInflate(XmlPullParser parser, View parent, Context context,
          AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {
      final int depth = parser.getDepth();
      int type;
      boolean pendingRequestFocus = false;
      while (((type = parser.next()) != XmlPullParser.END_TAG ||
              parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {
          if (type != XmlPullParser.START_TAG) {
              continue;
          }
          final String name = parser.getName();
          if (TAG_REQUEST_FOCUS.equals(name)) {
              pendingRequestFocus = true;
              consumeChildElements(parser);
          } else if (TAG_TAG.equals(name)) {
              parseViewTag(parser, parent, attrs);
          } else if (TAG_INCLUDE.equals(name)) {
              if (parser.getDepth() == 0) {
                  throw new InflateException("<include /> cannot be the root element");
              }
              parseInclude(parser, context, parent, attrs);
          } else if (TAG_MERGE.equals(name)) {
              throw new InflateException("<merge /> must be the root element");
          } else {
              final View view = createViewFromTag(parent, name, context, attrs);
              final ViewGroup viewGroup = (ViewGroup) parent;
              final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
              rInflateChildren(parser, view, attrs, true);
              viewGroup.addView(view, params);
          }
      }
      if (pendingRequestFocus) {
          parent.restoreDefaultFocus();
      }
      if (finishInflate) {
          parent.onFinishInflate();
      }
  }
  ```

  LayoutInflater将之前传进来的R.layout.scrren_title文件，封装成了XmlPullParser对象，然后在rInflate方法的中，对View树进行递归遍历，对其子级进行实例化。

  上述方法中调用到了ViewGroup的addView方法的

* 之后ViewGroup的addView方法，调用到了addViewInner方法：

  ```java
  private void addViewInner(View child, int index, LayoutParams params,
          boolean preventRequestLayout) {
      // tell our children
      if (preventRequestLayout) {
          child.assignParent(this);
      } else {
          child.mParent = this;
      }
  ```

  上述代码可以看出，直接将自身，赋值给了子View的mParent对象。

到了这里，基本就明白了：

* 普通View的mParent是它的父View

* DecorView的mParent是ViewRootImpl。因为DecorView是根View，已经没有再往上的父View了。

### 参考文章

* [关于View中mParent的来龙去脉](https://www.jianshu.com/p/a6fd2c4db80d)

