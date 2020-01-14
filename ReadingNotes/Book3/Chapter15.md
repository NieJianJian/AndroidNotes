## 第15章 插件化原理

### 1 插件化的产生

* 应用开发的痛点和瓶颈
  * 业务复杂，模块耦合
  * 应用间接入
  * 65536限制，内存占用大

**定义**：将一个应用按照插件的方式进行改造的过程就叫插件化。

***

### 2 插件化框架对比

插件化框架如下表：

| 插件化框架  | 作者 | 插件化框架       | 作者    |
| ----------- | ---- | ---------------- | ------- |
| DynamicAPK  | 携程 | dynamic-load-apk | 任玉刚  |
| DroidPlugin | 360  | Small            | Wequick |
| RePlugin    | 360  | VirtualApk       | 滴滴    |

VirtualApk在加载耦合插件方面是插件化框架的首选，具有普遍适用性。

***

### 3 Activity插件化

Activity插件化主要有3种实现方式，分别是**反射实现**、**接口实现**和**Hook技术实现**。

Hook技术实现主要有两种解决方案

* 通过Hook IActivityManager来实现。
* 通过Hook Instrumentation来实现。

### 3.1 Activity的启动过程

* 根Activity的启动过程

  <img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/rootactivitystart_processrelation.png" alt="根Activity的启动过程 " style="zoom:80%;" />

  * 首先Launcher进程向AMS请求创建根Activity
  * AMS判断根Activity所需的应用程序进程是否存在并启动
  * 不存在就会请求Zygote进程创建应用程序进程
  * 应用程序进程启动后，AMS会请求应用程序进程创建并启动根Activity

* 普通Activity的启动过程

  <img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/oridinaryactivity_startprocess.png" alt="普通Activity启动过程" style="zoom:80%;" />

  * 应用程序进程中的Activity向AMS请求创建普通Activity，AMS会对这个Activity的生命周期和栈进行管理，校验Activity等。
  * 如果Activity满足AMS校验，AMS就会请求应用程序进程中的ActivityThread去创建并启动普通Activity。

### 3.2 Hook IActivityManager方案实现

在AndroiManifest.xml中注册一个占坑Activity，用来通过AMS校验，以解决没有显式声明的问题。

* **1 注册Activity进行占坑**

  * TargetActivity用来代表已经加载进来的插件Activity，不需要在清单文件中注册。
  * StubActivity用来占坑，需要在清单文件中注册。

  MainActivity中直接启动TargetActivity肯定会报错（ActivityNotFoundException异常）。

* **2 使用占坑Activity通过AMS验证**

  为了防止报错，需要将启动的TargetActivity替换为StubActivity，用StubActivity来通过AMS验证。

  根据[Android7.0 AMS家族]([https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/Chapter6.md#11-android-70%E7%9A%84ams%E5%AE%B6%E6%97%8F](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/Chapter6.md#11-android-70的ams家族))和[Android8.0 AMS]([https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/Chapter6.md#12-android-80%E7%9A%84ams%E5%AE%B6%E6%97%8F](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/Chapter6.md#12-android-80的ams家族))家族来看，IActivitiManager都借助了Singleton类来实现单例。Singleton代码如下：

  ```java
  public abstract class Singleton<T> {
      private T mInstance;
      protected abstract T create();
      public final T get() {
          synchronized (this) {
              if (mInstance == null) {
                  mInstance = create();
              }
              return mInstance;
          }
      }
  }
  ```

  所以，IActivityManager是比较好的Hook点。

  * 由于Hook需要多次对字段进行反射操作，编写工具类FieldUtil：

    ```java
    public class FieldUtil {
        public static Object getField(Class clazz, Object target, String name) throws Exception {
            Field field = clazz.getDeclaredField(name);
            field.setAccessible(true);
            return field.get(target);
        }
        public static Field getField(Class clazz, String name) throws Exception{
            Field field = clazz.getDeclaredField(name);
            field.setAccessible(true);
            return field;
        }
        public static void setField(Class clazz, Object target, String name, Object value) throws Exception {
            Field field = clazz.getDeclaredField(name);
            field.setAccessible(true);
            field.set(target, value);
        }
    }
    ```

  * 接着定义替换IActivityManager的代理类IActivityManagerProxy，如下：

    ```java
    public class IActivityManagerProxy implements InvocationHandler {
        private Object mActivityManager;
        private static final String TAG = "IActivityManagerProxy";
        public IActivityManagerProxy(Object activityManager) {
            this.mActivityManager = activityManager;
        }
        @Override
        public Object invoke(Object proxy, Method method,Object[] args) throws Throwable{
            if ("startActivity".equals(method.getName())) { // 1
                Intent intent = null;
                int index = 0;
                for (int i = 0; i < args.length; i++) {
                    if (args[i] instanceof Intent) {
                        index = i;
                        break;
                    }
                }
                intent = (Intent) args[index];
                Intent stubIntent = new Intent(); // 2
                String packageNama = "com.nj.test";
                stubIntent.setClassName(packageNama, packageNama + ".StubActivity"); // 3
                stubIntent.putExtra(HookHelper.TARGET_INTENT, intent); // 4
                args[index] = stubIntent; // 5
            }
            return method.invoke(mActivityManager, args);
        }
    }
    ```

    * 拦截`startActivity`方法，获取参数`args`中第一个Intent对象，它原本要启动插件TargetActivity的Intent。
    * 新建`stubIntent`用来启动StubActivity。
    * 将TargetActivity的Intent保存到stubIntent中，以便于以后还原TargetActivity。
    * 将`stubIntent`赋值给参数`args`，这样启动目标就变成了StubActivity，通过也AMS校验。

  * 用代理类IActivityManagerProxy来替换IActivityManager，如下：

    ```java
    public class HookHelper {
        public static final String TARGET_INTENT = "target_intent";
        public static void hookAMS() throws Exception {
            Object defaultSingleton = null;
            if (Build.VERSION.SDK_INT >= 26) {//1
                Class<?> activityManageClazz =
                        Class.forName("android.app.ActivityManager");
                //获取activityManager中的IActivityManagerSingleton字段
                defaultSingleton = FieldUtil.getField(activityManageClazz,
                        null, "IActivityManagerSingleton");
            } else {
                Class<?> activityManagerNativeClazz =
                        Class.forName("android.app.ActivityManagerNative");
                //获取ActivityManagerNative中的gDefault字段
                defaultSingleton = FieldUtil.getField(activityManagerNativeClazz,
                        null, "gDefault");
            }
            Class<?> singletonClazz = Class.forName("android.util.Singleton");
            Field mInstanceField = FieldUtil.getField(singletonClazz, "mInstance");//2
            //获取iActivityManager
            Object iActivityManager = mInstanceField.get(defaultSingleton);//3
            Class<?> iActivityManagerClazz =
                    Class.forName("android.app.IActivityManager");
            Object proxy = Proxy.newProxyInstance(
                    Thread.currentThread().getContextClassLoader(),
                    new Class<?>[]{iActivityManagerClazz},
                    new IActivityManagerProxy(iActivityManager));
            mInstanceField.set(defaultSingleton, proxy);
        }
    }
    ```

    * 对系统版本进行区分，最终获取的是SIngleton< IActivityManager>类型的IActivityManagerSingleton或者gDefault字段。
    * 获取Singleton类中的`mInstance`字段。然后得到IActivityManagerSingleton或者gDefault字段中的T类型，T的类型为IActivityManager。
    * 最后动态创建代理类IActivityManagerProxy，用来替换IActivityManager。

  * 自定义一个Application，在其中调用`hookAMS`方法，如下：

    ```java
    public class MyApplication extends Application {
        @Override
        protected void attachBaseContext(Context base) {
            super.attachBaseContext(base);
            try {
                HookHelper.hookAMS();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
    ```

  * 在MainActivity中启动TargetActivity，如下：

    ```java
    public class MainActivity extends Activity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);
            findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    Intent intent = new Intent(MainActivity.this, TargetActivity.class);
                    startActivity(intent);
                }
            });
        }
    }
    ```

    点击button，启动的是StubAcitivity。说明通过了AMS的校验。

* **3 还原插件Activity**

  适用StubAcitivyt通过了AMS验证，但是还需要用插件TargetActivity来替换占坑的StubActivity。

  在[ActivityThread启动Activity的过程]([https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/Chapter4.md#13-activitythread%E5%90%AF%E5%8A%A8activity%E7%9A%84%E8%BF%87%E7%A8%8B](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/Chapter4.md#13-activitythread启动activity的过程))中，ActivityThread会通过H类将代码的逻辑切换到主线程，并发送`LAUNCH_ACTIVITY`消息进行处理，最终会调用Activity的onCreate方法。

  那么在哪里替换呢？来看Handle的`dispatchMessage`方法：

  ```java
  public void dispatchMessage(Message msg) {
      if (msg.callback != null) {
          handleCallback(msg);
      } else {
          if (mCallback != null) {
              if (mCallback.handleMessage(msg)) {
                  return;
              }
          }
          handleMessage(msg);
      }
  }
  ```

  如果Handler的Callback类型的`mCallback`不为null，就会执行`mCallback`的handleMessage方法，因此，`mCallbac`可以作为Hook点，我们可以用自定义的Callback来替换mCallback：

  ```java
  public class HCallback implements Handler.Callback {
      public static final int LAUNCH_ACTIVITY = 100;
      Handler mHandler;
      public HCallback(Handler handler) {
          mHandler = handler;
      }
      @Override
      public boolean handleMessage(Message msg) {
          if (msg.what == LAUNCH_ACTIVITY) {
              Object r = msg.obj;
              try {
                  //得到消息中的Intent(启动SubActivity的Intent)
                  Intent intent = (Intent) FieldUtil.getField(r.getClass(), r, "intent");
                  //得到此前保存起来的Intent(启动TargetActivity的Intent)
                  Intent target = intent.getParcelableExtra(HookHelper.TARGET_INTENT);
                  //将启动SubActivity的Intent替换为启动TargetActivity的Intent
                  intent.setComponent(target.getComponent());
              } catch (Exception e) {
                  e.printStackTrace();
              }
          }
          mHandler.handleMessage(msg);
          return true;
      }
  }
  ```

  对消息`LAUNCH_ACTIVITY`进行拦截，将启动StubActivity替换为启动TargetActivity的Intent。

  接着在HookHelper中定义一个`hookHandler`方法，如下：

  ```java
  public static void hookHandler() throws Exception {
      Class<?> activityThreadClass =
              Class.forName("android.app.ActivityThread");
      Object currentActivityThread = FieldUtil.getField(activityThreadClass,
              null, "sCurrentActivityThread");//1
      Field mHField = FieldUtil.getField(activityThreadClass, "mH");//2
      Handler mH = (Handler) mHField.get(currentActivityThread);//3
      FieldUtil.setField(Handler.class, mH, "mCallback", new HCallback(mH));
  }
  ```

  最后在MyApplication的`attachBaseContext`方法中调用HookHelper的`hookHandler`方法。

  重新点击启动按钮，发现启动的是插件TargetActivity。

* **4 插件Activity的生命周期**

  Activity的`finish`方法可以触发Activity的生命周期变化

  ```java
  public void finish(int finishTask) {
      ...
      if (ActivityManager.getService().finishActivity(mToken,resultCode, ...)){
  }
  ```

  上述代码中调用了AMS的`finishActivity`方法，紧接着AMS通过ApplicationThread调用ActivityThread，ActivityThread向H类发送`DESTROY_ACTIVITY`类型消息，H类接收这个消息会执行`handleDestroyActivity`方法，随后又调用`performDestroyActivity`方法，如下所示：

  ```java
  ActivityClientRecord performDestroyActivity(IBinder token,
                                              boolean finishing, int configChanges,
                                              boolean getNonConfigInstance, String reason) {
      ActivityClientRecord r = mActivities.get(token); // 1
      Class<? extends Activity> activityClass = null;
      ...
          try {
              r.activity.mCalled = false;
              mInstrumentation.callActivityOnDestroy(r.activity); // 2
              ...
              } 
      mActivities.remove(token);
      return r;
  }
  ```

  * 注释1处通过IBinder类型的token来获取ActivityClientRecord。
  * 注释2处调用Instrumentation的`callActivityOnDestroy`方法来调用Activity的`onDestroy`方法，并传入`r.activity`。

  启动Activity的时候调用ActivityThread的`performLaunchActivity`方法，

  ```java
   private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
            ...
              java.lang.ClassLoader cl = appContext.getClassLoader();
              activity = mInstrumentation.newActivity(
                      cl, component.getClassName(), r.intent);//1
             ...
               activity.attach(appContext, this, getInstrumentation(), r.token,
                          r.ident, app, r.intent, r.activityInfo, title, r.parent,
                          r.embeddedID, r.lastNonConfigurationInstances, config,
                          r.referrer, r.voiceInteractor, window, r.configCallback);
             ...
              mActivities.put(r.token, r);//2
             ...
          return activity;
      }
  ```

  * 注释1根据Activity的类名用ClassLoader加载Activity，
  * 接着调用Activity的`attach`方法，将`r.token`赋值给Activity的成员变量`mToken`。
  * 注释2处将ActivityClientRecord根据`r.token`保存在`mActivities`中。

  AMS和ActivityThread之间的通信采用了token来对Activity进行标识的。

  我们在Activity启动时用插件TargetActivity替换占坑StubActivity，这一过程在`performLaunchActivity`方法调用之前，因此注释2处的`r.token`指向的是TargetActivity，在`performDestroyActivity`的注释1处获取的就是代表TargetActivity的ActivityClientRecord。所以TargetActivity是具有生命周期的。

### 3.3 Hook Instrumentation方案实现

在Activity通过AMS校验前，会调用Activity的`startActivityForResult`方法：

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
                                   @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                        this, mMainThread.getApplicationThread(), mToken, this,
                        intent, requestCode, options);
        ...
    } else {
        ...
    }
}
```

在`startActivityForResult`方法中调用了Instrumentation的`execStartActivity`方法来激活Activity的生命周期。

在之前的ActivityThread中的`performLaunchActivity`方法的注释1处，调用了`newActivity`方法，其内部会用类加载器创建Activity的实例。所以，可以在Instrumentation的`execStartActivity`方法中用占坑StubActivity来通过AMS的验证，在Instrumentation的`newActivity`方法中还原TargetActivity。

自定义一个Instrumentation，在`execStartActivity`方法中将启动的TargetActivity替换为StubActivity，如下：

```java
public class InstrumentationProxy extends Instrumentation {
    private Instrumentation mInstrumentation;
    private PackageManager mPackageManager;
    public InstrumentationProxy(Instrumentation instrumentation,
                                PackageManager packageManager) {
        mInstrumentation = instrumentation;
        mPackageManager = packageManager;
    }
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token,
            Activity target, Intent intent, int requestCode, Bundle options) {
        List<ResolveInfo> infos = mPackageManager.queryIntentActivities(
                intent, PackageManager.MATCH_ALL);
        if (infos == null || infos.size() == 0) {
            intent.putExtra(HookHelper.TARGET_INTENsT_NAME,
                    intent.getComponent().getClassName());//1
            intent.setClassName(who,
                    "com.example.liuwangshu.pluginactivity.StubActivity");//2
        }
        try {
            Method execMethod = Instrumentation.class.getDeclaredMethod(
                    "execStartActivity", Context.class, IBinder.class,
                    IBinder.class, Activity.class, Intent.class, int.class, Bundle.class);
            return (ActivityResult) execMethod.invoke(mInstrumentation,
                    who, contextThread, token, target, intent, requestCode, options);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```

首先查找要启动的Activity是否已经在清单文件中注册了，如果没有就在注释1处将要启动的Activity（TargetActivity）的ClassName保存起来用于后面还原，接着在注释2处替换要启动的Activity为StubActivity，最后通过反射调用`execStartActvitiy`方法，这样就可以用StubActivity通过AMS验证。

在InstrumentationProxy中还原TargetActivity，如下：

```java
public Activity newActivity(ClassLoader cl, String className, Intent intent) throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    String intentName = intent.getStringExtra(HookHelper.TARGET_INTENT_NAME);
    if (!TextUtils.isEmpty(intentName)) {
        return super.newActivity(cl, intentName, intent);
    }
    return super.newActivity(cl, className, intent);
}
```

在newActivity方法中创建了此前保存的TargetActivity，完成了还原TargetActivity。

在HookHelper中编写`hookInstrumentation`方法，用于替换`mInstrumentaion`：

```java
public static void hookInstrumentation(Context context) throws Exception {
    Class<?> contextImplClass = Class.forName("android.app.ContextImpl");
    Field mMainThreadField  =FieldUtil.getField(contextImplClass,
            "mMainThread");//1
    Object activityThread = mMainThreadField.get(context);//2
    Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
    Field mInstrumentationField=FieldUtil.getField(activityThreadClass,
            "mInstrumentation");//3
    FieldUtil.setField(activityThreadClass,activityThread, "mInstrumentation",
            new InstrumentationProxy((Instrumentation) mInstrumentationField.get(activityThread),
            context.getPackageManager()));
}
```

注释1处获取ContextImpl类的ActivityThread类型的`mMainThread`字段。

注释2处获取当前上下文环境的ActivityThread对象。

注释3处获取ActivityThread类中的`mInstrumentation`字段，最后用InstrumentationProxy来替换`mInstrumentation`。

在MyApplication的`attachBaseContext`方法中调用`hookInstrumentation`方法。

***