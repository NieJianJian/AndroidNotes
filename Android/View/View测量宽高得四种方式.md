## View测量宽高得四种方式

View的大小是在layout阶段确定的，几乎所有情况下View的测量大小和最终大小是相等的，除非是重写View的layout方法手动更改了大小。一个好的习惯是在onLayout方法中获取View的测量宽高或最终宽高。

在Activity的生命周期中均无法保证正确的获得View的宽高信息，因为View的measure过程和Activity的生命周期方法不是同步的。因此无法保证调用执行某一生命周期的时候，View就一定测量完了。如果View还没有测量完成，那么获得的宽高就是0，这里给出以下四种解决办法。

* Activity/View#onWindowFocusChanged方法

  onWindowFocusChanged的含义是：View已经初始化完毕了，宽高已经准备好了，这个时候去获取宽高是没有问题的。不过onWindowFocusChanged会被调用多次，当Activity失去或者获得焦点，都会执行，比如频繁的进行onResume和onPause，那么onWindowFocusChanged也会频繁调用。

  ```java
  @Override
  public void onWindowFocusChanged(boolean hasFocus) {
      super.onWindowFocusChanged(hasFocus);
      if (hasFocus) {
          Log.i("niejianjian", "方法一 -> width -> " + mButton.getMeasuredWidth());
          Log.i("niejianjian", "方法一 -> height -> " + mButton.getMeasuredHeight());
      }
  }
  ```

* 第二种

  通过post将一个runnable投递到消息队列尾部，然后等待Looper调用runnable的时候，View也已经初始化好

  ```java
  private void getViewSizeMethod2() {
      mButton.post(new Runnable() {
          @Override
          public void run() {
              Log.i("niejianjian", "方法二 -> width -> " + mButton.getMeasuredWidth());
              Log.i("niejianjian", "方法二 -> height -> " + 					mButton.getMeasuredHeight());
          }
      });
  }
  ```

* 第三种

   ViewTreeObserver的众多回调都可以完成这个功能，比如使用OnGlobalLayoutListener这个接口，当View树的状态发生改变或者View树内部的View的可见性发生改变时，onGlobalLayout方法将被回调，因此这是获取View宽高一次很好的时机，需要注意的是，伴随着View树的状态该改变等，onGlobalLayout会被调用很多次。

  ```java
  private void getViewSizeMethod3() {
      ViewTreeObserver observer = mButton.getViewTreeObserver();
      observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
          @Override
          public void onGlobalLayout() {
              // remove掉之后就只执行一次了
              mButton.getViewTreeObserver().removeGlobalOnLayoutListener(this);
              Log.i("niejianjian", "方法三 -> width -> " + mButton.getMeasuredWidth());
              Log.i("niejianjian", "方法三 -> height -> " + mButton.getMeasuredHeight());
          }
      });
  }
  ```

* 第四种

  通过手动给View进行measure来得到View的宽高，这种方法还要分情况，根据View的LayoutParams来分：
  1).match_parent ：直接放弃，无法measure出具体的宽高，根据View的measure规程，构造此种MeasureSpec需要知道parentSize，即父容器的剩余空间，而这个时间我们无法知道parentSize的大小，理论上是不可能测出View的大小。获得的宽高应该是包裹内容的宽高，无法获得实际的宽高。
  2).具体的数值（dp/px）
  3).wrap_content :可以成功

  ```java
  private void getViewSizeMethod4() {
      mButton.measure(View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED),
              View.MeasureSpec.makeMeasureSpec(0, View.MeasureSpec.UNSPECIFIED));
  //    mButton.measure(0, 0); // 和上面等价
      Log.i("niejianjian", "方法四 -> width -> " + mButton.getMeasuredWidth()); // 0
      Log.i("niejianjian", "方法四 -> height -> " + mButton.getMeasuredHeight()); // 0
  }
  ```

