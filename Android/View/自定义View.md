## 自定义View

一般可以认为自定义View可以划分为3大类：

* 自定义View

  自定义View又分为：继承View、继承系统控件（如TextView）

* 自定义ViewGroup

  自定义ViewGroup又分为：继承ViewGroup、继承系统特性ViewGroup（如LinearLayout）

* 自定义组合控件

### 1. 自定义View须知

自定义View时，通常需要重写onDraw方法来绘制View的显示内容。

1. 让View支持wrap_content

   这是因为直接继承View或者ViewGroup的控件，xml布局中指定wrap_content的效果和match_parent的效果一致，所以需要在onDraw方法中对wrap_content进行特殊处理。

2. 如果有必要，让你的View支持padding

   直接继承View的控件，如果不在draw方法中处理padding，那么padding属性是无法起作用的。

   直接继承ViewGroup的控件需要在onMeasure和onLayout中考虑padding和子元素的margin对其的影响，不然将导致padding和margin失效。

3. 尽量不要在View中使用Handler，没必要

   View内部本身提供了post系列的方法，完全可以提到Handler的作用。

4. View中如果有线程或者动画，需要及时停止，参考View#onDetachedFromWindow

   如果线程或者动画，想要停止时，onDetachedFromWindow是一个很好的时机。当包含此View的Activity退出或者当前View被remove掉时，View的onDetachedFromWindow将会被调用，和此方法对应的是onAttachedToWindow，当包含此View的Activity启动时，View的onAttachedToWindow会被调用。同时View变得不可见时我们也需要停止线程和动画，防止内存泄露。

5. View带有滑动嵌套，需要处理滑动冲突

