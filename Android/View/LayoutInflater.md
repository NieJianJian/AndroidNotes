## 获取View实例——LayoutInflater

**注：本文基于源码Android 9.0（API 28**）

### 目录

1. LayoutInflater使用场景
2. 获取LayoutInflater实例的方法和原理
3. 通过inflate方法创建View实例
4. inflate原理分析
5. 拓展知识

***

### 1. LayoutInflater使用场景

LayoutInflater我们都用到过，比如在RecyclerView中加载一个View：

```java
public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
    View view = LayoutInflater.from(mContext).inflate(R.layout.fruit_item, parent, false);
    ViewHolder holder = new ViewHolder(view);
    return holder;
}
```

根据上面的内容，我们可以明白，LayoutInflater会通过inflate方法会创建一个View并返回。

LayoutInflater的作用和findViewById类似，不同的是findViewById是根据查找xml布局内id来返回相应的widget控件（如Button、TextView）。LayoutInflater的inflate方法是将xml文件转换成View。

比如我们的setContentView：

```java
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
}
```

还可以这样写：

```java
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    View view = LayoutInflater.from(this).inflate(R.layout.activity_main,
            (ViewGroup) findViewById(android.R.id.content), false);
    setContentView(view);
}
```

为什么可以这样写？我们来看下setContentView的源码：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

getWindow得到的了PhoneWindow，所以调用PhoneWindow的setContentView，我们继续往下看：

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
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
    ...
}
```

在PhoneWindow的setContentView中，就调用了Layout.inflate方法。

### 2. 获取LayoutInflater实例的方法和原理

获得LayoutInflater实例的方式有以下三种：

```java
LayoutInflater inflater = getLayoutInflater();
```

    LayoutInflater inflater = LayoutInflater.from(this);
```java
LayoutInflater inflater2 = (LayoutInflater) 
        getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

其实这三种方式的本质是相同的

Activity的getLayoutInflater()方法：

```java
public LayoutInflater getLayoutInflater() {
    return getWindow().getLayoutInflater();
}
```

然后是PhoneWindow的getLayoutInflater方法：

```java
public LayoutInflater getLayoutInflater() {
    return mLayoutInflater;
}
```

那mLayoutInflater是什么时候赋值的呢？来看PhoneWindow的构造方法：

```java
public PhoneWindow(Context context) {
    super(context);
    mLayoutInflater = LayoutInflater.from(context);
}
```

所以看出，Activity的getLayoutInflater()最终调用的就是第二种方法。

接着看LayoutInflater.from方法：

```java
public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

根据上述代码分析，不管哪种方法最终都是通过getSystemService方法获取的。所谓的context其实是ContextImpl实例，所以调用的是ContextImpl的getSystemSerivice方法：

```java
public Object getSystemService(String name) {
    return SystemServiceRegistry.getSystemService(this, name);
}
```

接下来看SystemServiceRegistry的getSystemService方法

```java
public static Object getSystemService(ContextImpl ctx, String name) {
    ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
    return fetcher != null ? fetcher.getService(ctx) : null;
}
```

其实就是从SYSTEM_SERVICE_FETCHERS中获取服务，其实它是一个HashMap结构：

```java
private static final HashMap<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
        new HashMap<String, ServiceFetcher<?>>();
```

那这个服务是什么在哪里存进去的呢？来看SystemServiceRegistry的registerService方法：

```java
private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
    SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
    SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
}
```

就是在这里put进去的，那registerService是什么时候调用的呢？

```java
final class SystemServiceRegistry {
    ...
    private SystemServiceRegistry() { }
    static {
        registerService(Context.ACCESSIBILITY_SERVICE, AccessibilityManager.class,
                new CachedServiceFetcher<AccessibilityManager>() {
            @Override
            public AccessibilityManager createService(ContextImpl ctx) {
                return AccessibilityManager.getInstance(ctx);
            }});
        registerService(Context.LAYOUT_INFLATER_SERVICE, LayoutInflater.class,
                new CachedServiceFetcher<LayoutInflater>() {
            @Override
            public LayoutInflater createService(ContextImpl ctx) {
                return new PhoneLayoutInflater(ctx.getOuterContext());
            }});
         ...
```

在SystemServiceRegistry的代码块中，会调用registerService，会创建一百多个服务。我们看到，LayoutInflater是通过PhoneLayoutInflater创建出来的。我们来看它的代码：

```javac
public class PhoneLayoutInflater extends LayoutInflater {
    private static final String[] sClassPrefixList = {
        "android.widget.",
        "android.webkit.",
        "android.app."
    };
    public PhoneLayoutInflater(Context context) {
        super(context);
    }
    protected PhoneLayoutInflater(LayoutInflater original, Context newContext) {
        super(original, newContext);
    }
    @Override protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
        for (String prefix : sClassPrefixList) {
            try {
                View view = createView(name, prefix, attrs);
                if (view != null) {
                    return view;
                }
            } catch (ClassNotFoundException e) {
            }
        }
        return super.onCreateView(name, attrs);
    }
    public LayoutInflater cloneInContext(Context newContext) {
        return new PhoneLayoutInflater(this, newContext);
    }
}
```

核心方法就是onCreateView方法，该方法通过将传递过来的View前面加上`"android.widget."`，

`"android.webkit."`，`"android.app."`用来得到该内置View对象的完整路径，最后根据路径来创建出对应的View。

### 3. 通过inflate方法创建View实例

通过LayoutInflater的inflate方法可以获取View实例，infalte有四个重载的形式，如下：

```java
public View inflate(@LayoutRes int resource, ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

```java
public View inflate(XmlPullParser parser, ViewGroup root) {
    return inflate(parser, root, root != null);
}
```

```java
public View inflate(@LayoutRes int resource, ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }
    final XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

以上三种方式，最终都会调用到第四种：

```java
public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot) {
```

* XmlPullParser parser：我们传过来的resourceid被转化成XmlPullParser对象，用于解析布局。
* ViewGroup root：这个传进的是一个ViewGroup对象，当attachToRoot为true的时候，root就是当前inflate方法返回view的父view，当attachToRoot为false的时候，root就是一个普通的view，用来帮助决定inflate方法返回view的LayoutParams，这块不理解的话，可以看下view的绘制流程，子view的大小是由父view的measurespec和子view的宽高值决定的。
* boolean attachToRoot，这个就是上边说的，用来决定inflate返回的view跟root是否存在父子布局关系。true表示存在父子关系，系统会将inflate方法返回的view添加到root的子view里。false不添加。

三个参数都明白了，我们在使用inflate方法时，parser参数是保证要传递的，因为我们要新建一个view保证是要有布局支撑的，root可以为null，但是如果为null就代表inflate返回的view没有容器支撑，那么parser布局你写的宽高等数值可能不会起作用，所以必须传一个root容器，attachToRoot当你想创建的view直接为root的子view就设置为true，如果你想创建的view只是需要一个容器，那么就设置attachToRoot为false。

View也有inflate方法，但实际上调用的也是LayoutInflater：

```java
public static View inflate(Context context, @LayoutRes int resource, ViewGroup root) {
    LayoutInflater factory = LayoutInflater.from(context);
    return factory.inflate(resource, root);
}
```

### 4. inflate原理分析

```java
private static final String TAG_MERGE = "merge";
private static final String TAG_INCLUDE = "include";
private static final String TAG_1995 = "blink";
private static final String TAG_REQUEST_FOCUS = "requestFocus";
private static final String TAG_TAG = "tag";
private static final String ATTR_LAYOUT = "layout";
```

先来看一下inflate的源码：

```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        ...
        View result = root;
        try {
            ...
            final String name = parser.getName();
            // 判断节点是不是 merge
            if (TAG_MERGE.equals(name)) { // 1
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                rInflate(parser, root, inflaterContext, attrs, false); // 2
            } else {
                // 创建xml中找到的根视图view
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);//3
                ViewGroup.LayoutParams params = null;
                if (root != null) { // 4
                    // 根据root创建params，（如果有的话）
                    params = root.generateLayoutParams(attrs); // 5
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }
                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true); // 6
                // 将xml视图的view附加到root上
                if (root != null && attachToRoot) { // 7
                    root.addView(temp, params); 
                }
                // 这是唯一改变result的地方，没有父root或者选择不附加到父root上，则直接返回xml的View
                if (root == null || !attachToRoot) {  // 8
                    result = temp;
                }
            }
        } catch (XmlPullParserException e) {
            ...
        } finally {
            ...
        }
        return result;
    }
}
```

根据上面的代码我们来分析做了些什么：

* 判断xml根节点是不是merge，如果是merge：
  * 父root为null或者attachToRoot选择了flase，则抛出异常，因为merge需要融合，而我们父root又为null或者选择不附加到父root，xml将不知何去何从，所以抛出异常。
  * 如果添加到父root的条件满足，则执行rInflate方法。
* xml的节点不是merge，即正常的布局，则调用createViewFromTag根据创建相应的根View，赋值给局部变量temp，这个有可能是我们最终返回的值（但需要一定的条件）。
* 为temp设置LayoutParams属性。
* 如果父root不为null并且选择附加到父root上，则将temp添加到父root上。
* 如果父root为null或者不选择附加到父root有一个条件满足，我们就返回temp。

我们来看下rInflate方法和rInflateChildren方法：

```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
        boolean finishInflate) throws XmlPullParserException, IOException {
    rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
}
```

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
            // 生成View
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            // 递归调用子集
            rInflateChildren(parser, view, attrs, true);
            // 添加到父view中
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

在rInflateChildren中其实也是调用了rInflate方法，rInflate方法是递归方法，用于递归xml层次结构并实例化视图，实例化其子集，然后调用onFinishInflate方法结束。

从rInflate方法中可以看出，我们的第二个参数parent也就是我们的也就是我们在inflate方法中调用rInflateChildren传进去的temp，也就是我们想要实例化的xml文件，在rInflate方法中，通过递归方法，一层一层的向下遍历xml树，根据节点创建相应的view，然后在一层一层的返回，将创建的view添加到父view，最终添加到temp，从而形成一个Dom树。

### 5. 拓展知识

我们根据上一节的inflate方法中，继续分析，我们在注释4处判断root是不是为null，不为null则根据root创建LayoutParams对象并且设置给temp，如果我们如下调用：

```java
View temp = getLayoutInflater().inflate(R.layout.test_layout, null);
Log.d("Test", "temp.getLayoutParams() == null:" + (temp.getLayoutParams() == null));
rootLayout.addView(tempLayout);
```

将root设置为null，然后我们判断temp.getLayoutParams的结果是什么呢？结果是null。

那么我们的布局将如何显示呢？我们来看ViewGroup的addView方法：

```java
public void addView(View child) {
    addView(child, -1);
}

public void addView(View child, int index) {
    if (child == null) {
        throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
    }
    LayoutParams params = child.getLayoutParams();
    if (params == null) {
        params = generateDefaultLayoutParams();
        if (params == null) {
            throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
        }
    }
    addView(child, index, params);
}
```

在addView方法中，我们判断LayoutParams为null，会调用generateDefaultLayoutParams生成默认的LayoutParams，我们来看该方法：

```java
protected LayoutParams generateDefaultLayoutParams() {
    return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}
```

会创建一个宽高都为WRAP_CONTENT的LayoutParams。但实际呢？

ViewGroup有很多子类，每个子类又有很大的不同，比如LinearLayout和RelativeLayout，所以，它们内部都重写了generateDefaultLayoutParams方法，并且有不同的实现：

```java
// LinearLayout
protected LayoutParams generateDefaultLayoutParams() {
    if (mOrientation == HORIZONTAL) {
        return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
    } else if (mOrientation == VERTICAL) {
        return new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.WRAP_CONTENT);
    }
    return null;
}
```

```java
// RelativeLayout
protected ViewGroup.LayoutParams generateDefaultLayoutParams() {
    return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}
```

根据上述我们看到了，不同的ViewGroup，会有不同的实现。

总结：

* root == null

  显示结果并不是我们xml设置的宽高， 将会在调用addView时根据实际的父布局创建对应的LayoutParams。

* root != null，attachToRoot == false

  xml设置的宽高生效，正常添加

* root != null，attachToRoot == true

  报错了：

  ```java
  Caused by: java.lang.IllegalStateException: The specified child already has a parent. You must call removeView() on the child's parent first.
  ```

  我们把rootLayout.addView(tempLayout);这一行删除掉就可以了，效果和第二种情况一样。

  因为我们在rInflate方法中有这样几行代码：

  ```java
  if (root != null && attachToRoot) {
      root.addView(temp, params); 
  }
  ```

  这这里就已经将xml文件实例化，并执行addView将其添加到root中，所以我们再次调用addView会报错。

### 参考文章

* [【Android源码】LayoutInflater 分析](https://www.jianshu.com/p/8ed7caedb911)
* [使用layoutinflater的正确姿势](https://blog.csdn.net/zq2114522/article/details/53002855)
* [LayoutInflater原理解析](https://blog.csdn.net/crazy1235/article/details/70175185)

