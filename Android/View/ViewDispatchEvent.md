# View 事件分发

### 参考链接

* [事件分发机制原理](https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/[12]Dispatch-TouchEvent-Theory.md)
* [事件分发机制详解](https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/[15]Dispatch-TouchEvent-Source.md)
* [可能是讲解Android事件分发最好的文章](https://www.jianshu.com/p/2be492c1df96)

***

> `√` 表示有该方法。
>
> `X` 表示没有该方法。

| 类型     | 相关方法             | Activity | ViewGroup | View |
| -------- | -------------------- | :------: | :-------: | :--: |
| 事件分发 | dispatchTouchEvent   |    √     |     √     |  √   |
| 事件拦截 | onIterceptTouchEvent |    X     |     √     |  X   |
| 事件消费 | onTouchEvent         |    √     |     √     |  √   |

### 1. Activity对点击事件的分发过程

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteractin();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent;
}
```

* 点击事件交给Activity所附属的Window进行分发。
* 如果所有的View都不消耗事件，Activity的onTouchEvent就会被调用。

Window类可以控制顶级View的外观和行为策略，它的唯一实现是PhoneWindow。PhoneWindow将事件直接传递给了根View——DecorView。

### 2. ViewGroup对点击事件的分发过程

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean result = false;             // 默认状态为没有消费过
    if (!onInterceptTouchEvent(ev)) {   // 如果没有拦截交给子View
        result = child.dispatchTouchEvent(ev);
    }
    if (!result) {                      // 如果事件没有被消费,询问自身onTouchEvent
        result = onTouchEvent(ev);
    }
    return result;
}
```

1. **ViewGroup是如何判断应该分配给哪一个childView的**？

   把所有的childView遍历一遍，如果手指触摸的点在childView区域就分配给这个View。

2. **当手指触摸的位置多个childView重叠时怎么分配**？

   当childView重叠时，默认分配给最上面的childView。后面加载的一般会覆盖掉之前的，所以显示在最上面的就是最后加载的。

3. **假设View1和View2重叠，并且View2遮挡住了View1**：

   * 只有View1是可点击的，事件将分配给View1，即使被View2遮挡。
   * 只有View2是可点击的，事件将分配给View2。
   * View1和View2均可点击时，事件将分配给View2。

   > 可点击：View注册了onClickListener、onLongClickListener、onContextListener等，或设置了`android:clickable="true"`，就代表这个View是可点击的。Button、CheckBox默认可点击。
   >
   > 给 View 注册 `OnTouchListener` 不会影响 View 的可点击状态。即使给 View 注册 `OnTouchListener` ，只要不返回 `true` 就不会消费事件。

4. **ViewGroup 和 ChildView 同时注册了事件监听器(onClick等)，哪个会执行**?

   ChildView先执行，因为onClick调用是在onTouchEvent中，childView的onTouchEvnet调用的ViewGroup，所以childView先执行。

5. View只有消费了ACTION_DOWN事件，才能接收到后续的MOVE和UP事件

6. 如果childView收到了消费了DOWN事件，之后被上层拦截了，将会收到一个ACTION_CANCEL。

7. 如果ViewGroup消费了DOWN事件，也就是onInterceptTouchEvent返回了true，该方法就不会再调用。

接下来具体分析下dispatchTouchEvent方法。

```java
`FLAG_DISALLOW_INTERCEPT`
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // 由于应用程序切换、ANR或某些状态改变，可能已经放弃或取消了上一个手势。进行重置。
    cancelAndClearTouchTargets(ev);
    resetTouchState(); // 该方法中会对FLAG_DISALLOW_INTERCEPT进行重置。
}

// 检查是否需要拦截.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
    if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);	// 询问是否拦截
        ev.setAction(action); 						// 恢复操作，防止被更改
    } else {
        intercepted = false;
    }
} else {
    // 没有目标来处理该事件，而且也不是一个新的事件事件(ACTION_DOWN), 进行拦截。
    intercepted = true;
}
```

上述代码可以看出，ViewGroup在两种情况下会**判断是否需要拦截当前事件**：事件类型为`ACTION_DOWN`或者`mFirstTouchTarget != null`。

* 当事件由ViewGroup的子元素成功处理时，`mFirstTouchTarget`会被赋值并执行子元素，即不为null；
* 如果当前ViewGroup拦截事件时，`mFirstTouchTarget != null`就不成立了。
* 由于ViewGroup拦截了ACTON_DOWN事件，(actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null)的判断条件就不成立了。将会导致ViewGroup的onInterceptTouchEvent不会在被调用。这就是之前说的，**为什么onInterceptTouchEvent返回了true就不会再调用**。

有一个特殊情况，那就是`FLAG_DISALLOW_INTERCEPT`标记位，通过`requestDisallowInterceptTouchEvnet`方法来设置，一般用于子View中。

* `FLAG_DISALLOW_INTERCEPT`被设置后，ViewGroup将无法拦截ACTION_DOWN以外的其他点击事件。
* 因为ACTION_DOWN会重置`FLAG_DISALLOW_INTERCEPT`标记位，将导致子View中设置的这个标记位无效。
* 所以当面对ACTION_DOWN事件时，ViewGroup总会调用自己的onInterceptTouchEvnet方法来询问是否要拦截事件。

`FLAG_DISALLOW_INTERCEPT`这个标识位的作用是让ViewGroup不再拦截事件，前提是ViewGroup没有拦截ACTION_DOWN，也就是ACTION_DOWN已经被子View消费。

接下来看看事件是如何向下分发由它的子View进行处理的：

```java
final View[] children = mChildren;
for (int i = childrenCount - 1; i >= 0; i--) {
    final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
    final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
    if (!canViewReceivePointerEvents(child)
            || !isTransformedTouchPointInView(x, y, child, null)) {
        ev.setTargetAccessibilityFocus(false);
        continue;
    }
    newTouchTarget = getTouchTarget(child);
    if (newTouchTarget != null) {newTouchTarget.pointerIdBits |= idBitsToAssign;
        break;
    }
    resetCancelNextUpFlag(child);
    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
        // Child wants to receive touch within its bounds.
        mLastTouchDownTime = ev.getDownTime();
        if (preorderedList != null) {
            // childIndex points into presorted list, find original index
            for (int j = 0; j < childrenCount; j++) {
                if (children[childIndex] == mChildren[j]) {
                    mLastTouchDownIndex = j;
                    break;
                }
            }
        } else {
            mLastTouchDownIndex = childIndex;
        }
        mLastTouchDownX = ev.getX();
        mLastTouchDownY = ev.getY();
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }
}
```

* 首先遍历ViewGroup的所有子元素，然后判断子元素是否能够接收到点击事件。

* 能否接收到到点击事件的衡量标准：

  * 子元素是否是Visible或者正在播放动画；
  * 点击事件的坐标是否落在子元素的区域内。

* dispatchTransformedTouchEvent实际上调用的是子元素的dispatchTouchEvent方法：

  ```java
  if (child == null) {
      handled = super.dispatchTouchEvent(event);
  } else {
      handled = child.dispatchTouchEvent(event);
  }
  ```

* 一旦子元素的dispatchTouchEvent返回true，表示消费此事件时，addTocuhTarget方法中，就会对`mFirstTouchTarget`进行赋值，并且跳出for循环。

  ```java
  private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
      final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
      target.next = mFirstTouchTarget;
      mFirstTouchTarget = target;
      return target;
  }
  ```

如果遍历所有的子元素后事件都没被消费，一般有两种情况：

* ViewGroup没有子元素；
* 子元素处理了点击事件，但是dispatchTouchEvent返回false，一般是因为子元素onTouchEvent返回false。

这两种情况下，ViewGroup会自己处理点击事件:

```java
if (mFirstTouchTarget == null) {
    handled = dispatchTransformedTouchEvent(ev, canceled, null,TouchTarget.ALL_POINTER_IDS);
}
```

上面这段代码中，第三个参数child为null，从之前分析看出，它会调用

```java
handled = super.dispatchTouchEvent(event);
```

这里看出，最终转到了View的dispatchTouchEvent方法中，交由View来处理。也就是把当前ViewGroup当成是一个View来处理。

### 3. View对点击事件的分发过程

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...
    boolean result = false;	// result 为返回值，主要作用是告诉调用者事件是否已经被消费。
    if (onFilterTouchEventForSecurity(event)) {
        ListenerInfo li = mListenerInfo;
        /** 
         * 如果设置了OnTouchListener，并且当前 View 可点击，就调用监听器的 onTouch 方法，
         * 如果 onTouch 方法返回值为 true，就设置 result 为 true。
         */
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        /** 
         * 如果 result 为 false，则调用自身的 onTouchEvent。
         * 如果 onTouchEvent 返回值为 true，则设置 result 为 true。
         */
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    ...
    return result;
}
```

简单一点的伪代码就是：

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    if (mOnTouchListener.onTouch(this, event)) {
        return true;
    } else if (onTouchEvent(event)) {
        return true;
    }
    return false;
}
```

由上述代码可以看出OnTouchListener的优先级高于onTouchEvent。

整体事件的优先级：**`onTouchListener > onTouchEvent > onLongClickListener > onClickListener`**。

onClick和onLongClick的具体调用位置在onTouchEvent：

```java
public boolean onTouchEvent(MotionEvent event) {
    ...
    final int action = event.getAction();
  	// 检查各种 clickable
    if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                ...
                removeLongPressCallback();  // 移除长按
                ...
                performClick();             // 检查单击
                ...
                break;
            case MotionEvent.ACTION_DOWN:
                ...
                checkForLongClick(0);       // 检测长按
                ...
                break;
            ...
        }
        return true;                        // ◀︎表示事件被消费
    }
    return false;
}
```

### 4. 总结

* ViewGroup默认不拦截任何事件。onInterceptTouchEvent方法默认返回false。
* View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击的（clickable和longClickable同时为false）。
* View的enable属性默认不影响onTouchEvent的默认返回值。

### 5. 案例分析

接下来根据根据具体场景来分析：

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/viewtouch.png" alt="图：View点击分析图 " align="left" />

上图中，RootView和ViewGroup1都是一个ViewGroup，View1是一个View。

1. **点击View1，并且View1是不可点击状态**，事件分发流程log如下：

   ```java
   RootView -> dispatchTouchEvent -> ACTION_DOWN
   RootView -> onInterceptTouchEvent -> ACTION_DOWN
   ViewGroup1 -> dispatchTouchEvent -> ACTION_DOWN
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_DOWN
   View1 -> dispatchTouchEvent -> ACTION_DOWN
   View1 -> onTouchEvent -> ACTION_DOWN
   ViewGroup1 -> onTouchEvent -> ACTION_DOWN
   RootView -> onTouchEvent -> ACTION_DOWN
   ```

   * 由于View1是不可点击的，并且默认没有View消费这个事件。
   * 事件传递顺序为RootView -> ViewGroup1 -> View1。
   * 事件没有被消费掉，最终会回传到Activity，然后被抛弃。
   * 由于没有View消费ACTION_DOWN事件，所以也就不会接收到后续的事件序列。

2. **点击View1，并且View1是可点击的状态**， 也就是View1要消费这个事件，分发流程如下：

   ```java
   RootView -> dispatchTouchEvent -> ACTION_DOWN
   RootView -> onInterceptTouchEvent -> ACTION_DOWN
   ViewGroup1 -> dispatchTouchEvent -> ACTION_DOWN
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_DOWN
   View1 -> dispatchTouchEvent -> ACTION_DOWN
   View1 -> onTouchEvent -> ACTION_DOWN
   
   RootView -> dispatchTouchEvent -> ACTION_UP
   RootView -> onInterceptTouchEvent -> ACTION_UP
   ViewGroup1 -> dispatchTouchEvent -> ACTION_UP
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_UP
   View1 -> dispatchTouchEvent -> ACTION_UP
   View1 -> onTouchEvent -> ACTION_UP
   ```

   * 由于View消费了ACTION_DOWN事件，所以会接收到后续的事件序列。
   * 如果中间有ACTION_MOVE事件，同样从RootView传递到View1。

3. **点击View1，并且View1是可点击的状态，并且滑动到View1外**，分发流程如下：

   ```java
   RootView -> dispatchTouchEvent -> ACTION_DOWN
   RootView -> onInterceptTouchEvent -> ACTION_DOWN
   ViewGroup1 -> dispatchTouchEvent -> ACTION_DOWN
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_DOWN
   View1 -> dispatchTouchEvent -> ACTION_DOWN
   View1 -> onTouchEvent -> ACTION_DOWN
   
   RootView -> dispatchTouchEvent -> ACTION_MOVE
   RootView -> onInterceptTouchEvent -> ACTION_MOVE
   ViewGroup1 -> dispatchTouchEvent -> ACTION_MOVE
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_MOVE
   View1 -> dispatchTouchEvent -> ACTION_MOVE
   View1 -> onTouchEvent -> ACTION_MOVE
   
   ... 省略多组ACTION_MOVE事件
     
   RootView -> dispatchTouchEvent -> ACTION_UP
   RootView -> onInterceptTouchEvent -> ACTION_UP
   ViewGroup1 -> dispatchTouchEvent -> ACTION_UP
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_UP
   View1 -> dispatchTouchEvent -> ACTION_UP
   View1 -> onTouchEvent -> ACTION_UP
   ```

   * 即使滑动到了View1外围，View1照样会收到MOVE和UP事件。因为**某个View一旦决定拦截，那么这个事件序列都只能由它来处理**。

4. **点击View1，并且View1是可点击的状态， 手指滑动一段距离后，ViewGroup1进行了事件拦截**，也就是onInterceptTouchEvent返回了true，分发流程如下：

   ```java
   RootView -> dispatchTouchEvent -> ACTION_DOWN
   RootView -> onInterceptTouchEvent -> ACTION_DOWN
   ViewGroup1 -> dispatchTouchEvent -> ACTION_DOWN
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_DOWN
   View1 -> dispatchTouchEvent -> ACTION_DOWN
   View1 -> onTouchEvent -> ACTION_DOWN
     
   RootView -> dispatchTouchEvent -> ACTION_MOVE
   RootView -> onInterceptTouchEvent -> ACTION_MOVE
   ViewGroup1 -> dispatchTouchEvent -> ACTION_MOVE
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_MOVE
   View1 -> dispatchTouchEvent -> ACTION_MOVE
   View1 -> onTouchEvent -> ACTION_MOVE
     
   ... 省略多组ACTION_MOVE事件
     
   RootView -> dispatchTouchEvent -> ACTION_MOVE
   RootView -> onInterceptTouchEvent -> ACTION_MOVE
   ViewGroup1 -> dispatchTouchEvent -> ACTION_MOVE
   ViewGroup1 -> onInterceptTouchEvent -> ACTION_MOVE
   
   View1 -> dispatchTouchEvent -> ACTION_CANCEL
   View1 -> onTouchEvent -> ACTION_CANCEL
     
   RootView -> dispatchTouchEvent -> ACTION_MOVE
   RootView -> onInterceptTouchEvent -> ACTION_MOVE
   ViewGroup1 -> dispatchTouchEvent -> ACTION_MOVE
   ViewGroup1 -> onTouchEvent -> ACTION_MOVE
     
   ... 省略多组ACTION_MOVE事件
     
   RootView -> dispatchTouchEvent -> ACTION_UP
   RootView -> onInterceptTouchEvent -> ACTION_UP
   ViewGroup1 -> dispatchTouchEvent -> ACTION_UP
   ViewGroup1 -> onTouchEvent -> ACTION_UP
   ```

   * 一旦上层进行了拦截，之间接收DOWN事件的View1将会收到**ACTION_CANCEL**事件。
   * ViewGroup1在onInterceptTouchEvent返回了true，拦截了事件，后续的事件序列会传到ViewGroup1的onTouchEvent中，不会再传递给View1了。

5. **点击ViewGroup1，ViewGroup1不可点击**：

   * 和"点击View1，并且View1是不可点击状态"原理一致。

6. **点击ViewGroup1，并且ViewGroup1是可点击的状态**

   * 和"点击View1，并且View1是可点击的状态"原理一致。
   * ViewGroup一旦决定拦截，它的**onInterceptTouchEvent就不会再**被调用。

### 6. 事件拦截处理

#### 6.1 外部拦截法

由于事件的传递都是先经过父容器的拦截处理，如果父容器需要此事件就进行拦截，如果不需要此事件就不拦截。

外部拦截法需要重写父容器的onInterceptTouchEvent方法：

```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
    boolean intercept = false;
    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            intercept = false;
            break;
        case MotionEvent.ACTION_MOVE:
            if (父容器需要点击事件) {
                intercept = true;
            } else {
                intercept = false;
            }
            break;
        case MotionEvent.ACTION_UP:
            intercept = false;
            break;
    }
    return intercept;
}
```

* ACTION_DOWN，父容器必须返回false，如果一旦返回了true，后续的ACTION_MOVE和ACTION_UP都会直接交由父容器处理，就没法传递给子View了
* ACTION_MOVE事件可以根据需求决定是否要拦截。
* ACTION_UP必须返回false，因为对了父容器来说，本身没有什么意义，但是对于子元素来说，如果ACTION_DOWN和ACTION_MOVE都是子View处理，父容器并未拦截，而ACTION_UP缺返回了true，子View将无法接收到ACTION_UP事件。如果子View设置了onClick事件，将无法触发。

#### 6.2 内部拦截法

内部拦截法是指父容器默认不拦截任何事件，所有的事件都交由子View处理，如果子View需要此事件就直接消耗掉，否则就交由父容器处理。需要重写子View的dispatchTouchEvent方法：

```java
# 子View的dispatchTouchEvent
public boolean dispatchTouchEvent(MotionEvent ev) {
    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            parent.requestDisallowInterceptTouchEvent(true);
            break;
        case MotionEvent.ACTION_MOVE:
            if (父容器需要点击事件) {
                parent.requestDisallowInterceptTouchEvent(false);
            } 
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
    return super.dispatchTouchEvent(ev);
}
```

* `requestDisallowInterceptTouchEvent`我理解的字面意思是，请求禁止拦截点击事件：
  * 若为true，就是同意禁止拦截点击事件，也就是不拦截；
  * 若为false，就是不同意禁止拦截点击事件，双重否定表肯定，也就是拦截。

* 父容器不能拦截ACTION_DOWN事件，否则所有的事件都无法传递到子View中。
* 父容器也需要默认拦截除了ACTION_DOWN以外的其他事件，这样当子View调用parent.requestDisallowInterceptTouchEvent(false)时，父容器才能继续拦截所需的事件。

```java
# 父容器的onInterceptTouchEvent
public boolean onInterceptTouchEvent(MotionEvent ev) {
    int action = ev.getAction();
    if (action == MotionEvent.ACTION_DOWN) {
        return false;
    } else {
        return true;
    }
}
```

