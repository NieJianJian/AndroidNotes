## 动画

动画的分类：

* 帧动画

  一段时间内通过顺序播放一系列的图像从而产生的动画效果。

* 补间动画（View动画）

  通过对View空间进行旋转、渐变、平移、缩放等图像变换来产生动画效果。

  * 只能对View进行操作；
  * 四种动画，不具备扩展性；
  * 只是视觉上的改变，属性位置并未改变。

* 属性动画、

  原理是通过不断的修改对象的属性值来实现。

  * 不在只针对View，可以用于任意对象；
  * 不再仅限于旋转、渐变、平移、缩放这几种简单的动画操作；
  * 不再只是是视觉上的改变，可以改变属性。

使用动画的注意事项：

* 帧动画中，当图片数量较多且较大时容易出现OOM。
* 属性动画中有一类无限循环的动画，Activity退出时要即时停止，防止内存泄露。View动画不存在此问题。
* View动画是对影像做动画，无法真正改变View状态，因此有时会出现动画完成后View无法隐藏，即View.GONE失效，这时候只要调用view.clearAnimation()清除View动画即可解决问题。
* 使用动画过程中，尽量使用dp，不要使用px。
* 开启硬件加速可以提高动画流畅度。

***

### 1. 帧动画

* 使用xml定义

  新建文件`image_anim.xml`，放到drawable目录下：

  ```java
  <?xml version="1.0" encoding="utf-8"?>
  <animation-list xmlns:android="http://schemas.android.com/apk/res/android" android:oneshot="false">
  <item android:drawable="@drawable/uncheck_in1" android:duration="200"/>
  <item android:drawable="@drawable/uncheck_in2" android:duration="200"/>
  <item android:drawable="@drawable/uncheck_in3" android:duration="200"/>
  </animation-list>
  ```

  定义好之后使用方法如下：

  * 第一种使用方法

    ```java
    <ImageView
        android:id="@+id/imageview_anim"
        android:layout_height="wrap_content"
        android:layout_width="wrap_content"
        android:background="@drawable/image_anim"/>
    // Java代码中使用      
    ImageView imageView = (ImageView) findViewById(R.id.imageview_anim);
    AnimationDrawable animation = (AnimationDrawable)imageView.getBackground();
    animation.start();
    animation.stop();
    ```

  * 第二种使用方法

    ```java
    ImageView imageView = (ImageView) findViewById(R.id.imageview_anim);
    AnimationDrawable animation = (AnimationDrawable) getResources().
      		getDrawable(R.drawable.image_anim);
    imageView.setBackgroundDrawable(frameAnim);
    animation.start();
    animation.stop();
    ```

  * 第三张使用方法

    ```java
    ImageView imageView = (ImageView) findViewById(R.id.imageview_anim);
    mImageView.setBackgroundResources(R.drawable.image_anim);
    AnimationDrawable animation = (AnimationDrawable) imageView.getBackgorund();
    animation.start();
    animation.stop();
    ```

* 使用Java代码定义

  ```java
  imageView = findViewById(R.id.iv_frame1);
  animation = new AnimationDrawable();
  // 为AnimationDrawable添加动画帧
  animation.addFrame(getResources().getDrawable(R.drawable.uncheck_in1), 200);
  animation.addFrame(getResources().getDrawable(R.drawable.uncheck_in2), 200);
  animation.addFrame(getResources().getDrawable(R.drawable.uncheck_in3), 200);
  animation.setOneShot(false);
  imageView.setBackground(animation);
  animation.start();
  animation.stop();
  ```

### 2. 补间动画

在 /res 目录下新建 anim 文件夹，在 anim 文件夹下新建 xml 文件：

```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- 透明度变化，从 1 到 0-->
    <alpha
        android:duration="5000"
        android:fromAlpha="1.0"
        android:toAlpha="0.0"/>
    <!-- 缩放，宽高都是从 1 到 0
         pivotX、pivotY 代表缩放中心点的横竖坐标
         interpolator 代表动画模式，我设置为先加速、后减速-->
    <scale
        android:duration="5000"
        android:fromXScale="1.0"
        android:fromYScale="1.0"
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        android:pivotX="50%"
        android:pivotY="50%"
        android:toXScale="0.0"
        android:toYScale="0.0"/>
    <!-- 平移动画，从左上角到右下角-->
    <translate
        android:duration="5000"
        android:fromXDelta="150"
        android:fromYDelta="150"
        android:toXDelta="200"
        android:toYDelta="200"/>
    <!-- 旋转动画，fromDegrees 初始角度
          结束角度 toDegrees-->
    <rotate
        android:duration="5000"
        android:fromDegrees="0"
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        android:pivotX="50%"
        android:pivotY="50%"
        android:toDegrees="+360"/>
</set>
```

代码中给View添加动画

```java
ImageView iv_bee=findViewById(R.id.iv_bee);
Animation loadAnimation =AnimationUtils.loadAnimation(this,R.anim.part5view_anim1);
loadAnimation.setFillAfter(true);
iv_bee.startAnimation(loadAnimation);
```

用代码定义动画：

```java
AlphaAnimation alphaAnimation = new AlphaAnimation(0, 1);
alphaAnimation.setDuration(1000);
iv_bee.startAnimation(alphaAnimation);
```

### 3. 属性动画

**属性动画原理：属性动画要求动画作用的对象提供该属性的get和set方法，属性动画根据外界传递的该属性的初始值和最终值以及要经历的属性值，然后通过你过内部的计算方法计算出在各个时间段该属性的取值，以动画的效果多次去调用set方法更新该属性值，在动画运行周期内不断的计算、更新属性值，从而达到对象的属性动画效果**。

总结一下，如果对object的属性abc做动画，想让动画生效，要同时满足两个条件：

* object必须要提供setAbc方法，如果动画的时候没有传递初始值，那么还要提供getAbc方法，因为系统要去取abc属性的初始值（如果不满足，直接Crash）。
* object的setAbc对属性abc所做的改变必须能够通过某种方法反应出来，比如会带来UI的改变之类的（如果不满足，动画无效）。

首先，先将一种简单的属性动画：

```java
mImageView.animate().translationX(500);
```

View的animate()方法会返回一个ViewPropertyAnimator对象：

```java
public ViewPropertyAnimator animate() {
    if (mAnimator == null) {
        mAnimator = new ViewPropertyAnimator(this);
    }
    return mAnimator;
}
```

ViewPropertyAnimator就是一个属性动画，对View进行了封装，然后可以调用相关的平移、旋转、缩放和透明度等方法进行操作。还可以按照如下方法设置插值器：

```java
mImageView.animate().setInterpolator(new LinearInterpolator()).translationX(500);
```

这样的方法，不需要调用start方法就可以直接启动动画，但是确定就是，只能使用ViewPropertyAnimator的内置方法，不能对其他属性进行操作。接下来就讲讲我们常用的属性动画。

属性动画有以下三个常用的类：

* **ValueAnimator**——属性动画的核心类

  作用是在一定的时间段内不断修改对象的某个属性值。内部使用一种时间循环的机制来计算值与值之间的动画过渡。

  ```java
  ValueAnimator animator = ValueAnimator.ofFloat(0.0f , 1.0f);
  animator.setDuration(1000);
  animator.start();
  ```

* **ObjectAnimator**——对任意属性进行动画操作

  ObjectAnimator继承于ValueAnimtor。

  ValueAnimtor只能对值进行一个平滑的动画过度，ObjectAnimator可以直接对任意属性进行动画操作。

  它的原理是在初始时设置目标对象、目标属性以及要经历的属性值，然后通过内部的计算方式计算出在各个时间段该属性的取值，在动画运行期间通过目标对象属性的setter函数更新该属性值，如果该属性值没有setter属函数，那么将通过反射的形式更新目标属性值。在运行周期内不断的计算、更新新的属性值，从而达到对象的属性动画效果。

  ```java
  ObjectAnimator animator = ObjectAnimator.ofFloat(mView, "alpha", 0.0f , 1.0f);
  animator.setDuration(1000);
  animator.start();
  ```

  ObjectAnimator通过静态工厂来直接返回一个ObjectAnimator对象，

  * 参数1是需要操作的View对象；
  * 参数2是需要操作的属性；
  * 最后的参数是一个可变数组，起始值和目标值，以及中间的多个过渡值。

* **AnimatorSet**——实现丰富多彩的动画效果

  AnimatorSet可以将多个动画组合在一起执行。

  ```java
  ObjectAnimator animator1 = ObjectAnimator.ofFloat(mView, "alpha", 0.0f , 1.0f);
  ObjectAnimator animator2 = ObjectAnimator.ofFloat(mView, "scaleX", 1f , 0.5f);
  ObjectAnimator animator3 = ObjectAnimator.ofFloat(mView, "translationX", 300);
  AnimatorSet set = new AnimatorSet();
  set.setDuration(1000);
  set.playTogether(animator1, animator2, animator3);
  set.start();
  ```

* **插值器和估值器**

  * **TimeInterpolator**：时间插值器，作用是根据时间流失的百分比来计算出当前属性值改变的百分比。
    * LinearInterpolator：线性插值器，匀速动画；
    * AccelerateDecelerateInterpolator：加速减速插值器，动画两头慢中间快；
    * DecelerateInterceptor：减速插值器，动画越来越慢。
    * AccelerateInterpolator：加速插值器，动画越来越快；
  * **TypeEvaluator**：类型估值算法，作用是根据当前属性改变的百分比来计算改变后的属性值。
    * IntEvaluator：针对整型属性；
    * FloatEvaluator：针对浮点型属性；
    * ArgbEvaluator：针对Color属性。

  TypeEvaluator只有一个evaluator方法，该函数作用就是计算出新的属性值：

  ```java
  public interface TypeEvaluator<T> {
      /**
       * @param fraction   已执行时间占总时间的百分比
       * @param startValue 属性的起始值
       * @param endValue   属性的最终值
       */
      public T evaluate(float fraction, T startValue, T endValue);
  }
  ```

  整型估值算法的源码：

  ```java
  public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
      int startInt = startValue;
      return (int)(startInt + fraction * (endValue - startInt));
  }
  ```

  TimeInterpolater的作用是用来修改fraction的值，它只有一个getInterpolatiron方法：

  ```java
  public interface TimeInterpolator {
  
      /**
       * @param 参数就是fraction本身，一个0.0到1.0之间的值
       * @return 返回值是修改后的fraction
       */
      float getInterpolation(float input);
  }
  ```

  例如线性插值器是匀速执行的，所以它没有修改fraction的值：

  ```java
  public class LinearInterpolator extends BaseInterpolator implements NativeInterpolatorFactory {
      public LinearInterpolator() { }
      public LinearInterpolator(Context context, AttributeSet attrs) { }
      public float getInterpolation(float input) {
          return input;
      }
  }
  ```

  例如加速插值器是对fraction参数进行乘方：

  ```java
  public float getInterpolation(float input) {
      return input * input;
  }
  ```

* 属性动画的监听器

  AnimatorListener监听动画的开始、结束、取消以及重复播放。

  ```java
  public static interface AnimatorListener {
      void onAnimationStart(Animator animation);
      void onAnimationEnd(Animator animation);
      void onAnimationCancel(Animator animation);
      void onAnimationRepeat(Animator animation);
  }
  ```

  AnimatorUpdateListener监听整个动画过程，每播放一帧onAnimationUpdate就会被调用一次。

  ```java
  public static interface AnimatorUpdateListener {
      void onAnimationUpdate(ValueAnimator animation);
  }
  ```

* 对任意属性做动画

  提出一个问题：给Button加一个动画，让Button的宽度从100px增加到500px。

  * 假设用View动画的scaleX，对x方法进行缩放，可以让Buttonx方向变大，看起来好像宽度增加，实际上是Button被方法而已，而且背景以及文本都会被拉伸，甚至可能超出屏幕。

  * 属性动画尝试

    ```java
    ObjectAnimator.ofInt(mButton, "width", 500).setDuration(5000).start();
    ```

    但是发现没有效果，而且Button内部是提供了getWidth和setWidth方法的。

    分析源码发现，Button继承了TextView，setWidth是TextView提供的方法：

    ```java
    public void setWidth(int pixels) {
        mMaxWidth = mMinWidth = pixels;
        mMaxWidthMode = mMinWidthMode = PIXELS;
        requestLayout();
        invalidate();
    }
    public final int getWidth() {
        return mRight - mLeft;
    }
    ```

    getWidth是可以获取View宽度的，但是setWidth的作用并不是设置View的高度，所以动画无效。也就是只满足了属性动画的条件1，没有满足条件2。

  * 我们可以有三种解决办法

    * 给你的对象加上get和set方法，如果你有权限的话；
    * 用一个类来包装原始对象，简介地为他们提供get和set方法；
    * 采用ValueAnimator，监听动画过程，自己实现属性的改变。

    1. 方案一往往是不可行的，因为我们没有权限

    2. 方案二是一个很有用的解决办法：

       ```java
       ViewWrapper wrapper = new ViewWrapper(mButton);
       ObjectAnimator.ofInt(wrapper, "width", 500).setDuration(5000).start();
       private static class ViewWrapper {
           private View mTarget;
           public ViewWrapper(View target) {
               this.mTarget = target;
           }
           public int getWidth() {
               return mTarget.getLayoutParams().width;
           }
           public void setWidth(int width) {
               mTarget.getLayoutParams().width = width;
               mTarget.requestLayout();
           }
       }
       ```

    3. 方案三，ValueAnimator本身不作用于任何对象，也就是说直接使用没有任何动画效果。

       ```java
       private void performAnimate(final View target, final int start, final int end) {
           ValueAnimator animator = ValueAnimator.ofInt(1, 100);
           animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
               private IntEvaluator mEvaluator = new IntEvaluator();
       
               @Override
               public void onAnimationUpdate(ValueAnimator animation) {
                   // 获取动画当前进度知，整型1 ~ 100之间
                   int currentValue = (int) animation.getAnimatedValue();
                   // 获取当前进度占整个动画过程的百分比，浮点型，0 ~ 1之间
                   float fraction = animation.getAnimatedFraction();
                   // 通过整型估值器来计算出宽度，设置给Button
                   target.getLayoutParams().width = 
                           mEvaluator.evaluate(fraction, start, end);
                   target.requestLayout();
               }
           });
           animator.setDuration(5000).start();
       }
       
       performAnimate(mButton, mButton.getWidth(), 500);
       ```

### 参考文献

1. [Android进阶之绘制 - 自定义View完全掌握系列一【HenCoder】](https://ke.qq.com/course/313640?taid=2323637436991784)