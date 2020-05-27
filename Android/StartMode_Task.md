## Activity的启动模式以及任务栈

### 1. Activity的启动模式

Activity的启动模式有四种，分别是：standard、singleTop、singleTask、singleInstance。它们的使用方法是在AndroidManifest中设置Activity的`android:launchMode`属性：

```xml
<activity android:name=".MainActivity" android:launchMode="singleTask" />
```

还可以通过Intent中设置标识位来为Activity指定启动模式：

```java
Intent intent = new Intent(this, SecondActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(intent);
```

当以上两种方式同时存在的时候，以第二种方式为准。

#### 1.1 四种启动模式介绍

* standard

  标准模式。每次都会创建新的实例，谁启动了这个Activity，那么这个Activity就运行在启动它的那个Activity所在的栈中。

* singleTop

  栈顶复用模式。在启动时判断要启动的Activity是否已经位于栈顶，如果是则不会创建新的Activity实例，同时它的onNewIntent方法会被回调；如果要启动的Activity已经存在与栈中，但不是位于栈顶，同样需要创建新的Activity实例。栈顶复用的生命周期调用链如下：

  ```
  onPause -> onNewIntent -> onResume
  ```

* singleTask

  栈内复用模式。只要启动的Activity在栈内存在，那么多次启动该Activity都不会创建新的实例，不管是不是位于栈顶。和singleTop一样会调用onNewIntent方法。singltTask还有一个特殊的属性，会将栈内位于该启动Activity之上的Activity全部销毁。

  上述的情况是同一个APP中启动这个singleTask的Activity，如果其他程序启动这个singleTask模式的Activity，那么会创建一个新的任务栈。

  如果启动模式为singleTask的Activity已经在一个后台任务栈中，那么启动后，后台的这个任务栈将被一起切换到前台。

* singleInstance

  单实例模式。会创建一个新的任务栈存放要启动的Activity，而且该任务栈中只存在这一个Activity。假设应用A的任务栈中创建了MainActivity的实例，且启动模式为singleInstance，如果应用B也要启动MainActivity，则不需要创建，两个应用共享该Activity实例。

#### 1.2 使用场景：

* standard：默认的启动模式，如果不指定Activity的启动模式，则使用这种方式启动Activity。
* singleTop：常用于详情页，比如微信接收到10条通知栏消息，我们不可能创建10个实例，所以采用singleTop
* singleTask：常用于Home主页面，不管在哪个页面返回到主页面，都cleanTop清除掉主页面之上的页面。也可用于退出整个应用：在要退出的Activity中转到启动模式为singleTask的主Activity，从而将主Activity之上的Activity都清除掉，然后重写主Activity的onNewIntent方法，然后调用finish()。
* singleInstance：常见于闹钟页面、来电页面。

#### 1.3 其他

1. 用ApplicationContext去启动Activity会报错，错误如下：

   ```
   android.util.AndroidRuntimeException: Calling startActivity() from outside of an Activity  context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?
   ```

   这是由于非Activity类型的Context（如ApplicationContext）并没有所谓的栈，所以需要为待启动的Activity指定`FLAG_ACTIVITY_NEW_TASK`标记位，这样启动的时候就可以创建一个新的任务栈。

2. 如果启动模式为singleTask的Activity已经在后台一个任务栈中，那么启动后，后台的这个任务栈将一起被切换到前台。

   假设目前有两个任务栈，前台任务栈的情况是AB（B在A之上），后台任务栈的情况为CD，并且CD的启动模式为singleTask。现在B请求启动D，那么整个后台任务栈都会被切换到前台，这时候整个任务栈变成了ABCD。当用户按back键的时候，列表的Activity会一一出栈。如果请求的是启动C，那么启动后的任务栈就会变为ABC。

#### 1.4 Intent FLAG 启动模式

* Intent.FLAG_ACTIVITY_NEW_TASK

  使用一个新的Task来启动一个Activity，通常使用在从Service中启动一个Activity的场景，因为在Service中并不存在Activity栈。严格意义上说，FLAG_ACTIVITY_NEW_TASK会判断你要启动的Activity的栈是否已经存在，如果存在，就会带到前台，并不会创建新的栈。如果不存在，才会创建新栈。

* FLAG_ACTIVITY_SINGLE_TOP

  等同于singltTop模式。

* FLAG_ACTIVITY_CLEAR_TOP

  等同于singleTask模式。

* FLAG_ACTIVITY_NO_HISTORY

  使用该模式启动Activity，当该Activity启动其他Activity后，该Activity就会消失，不会保留在栈中。如A启动B，B中以该模式启动了C，C再启动D，则当前Activity栈为ABD。

### 2. 任务栈

一个Android应用会拆分成多个Activity，每个Activity之间通过Intent进行了连接，而Android系统中通过栈结构来保存整个App的Activity，栈底的元素是整个任务的发起者。任务栈是一种"后进先出"的栈结构。当栈中没有Activity是，系统会回收这个任务栈。

当一个App启动时，如果当前系统不存在该App的任务栈，那么系统就会创建一个任务栈。此后，这个App所启动的Activity都将在这个任务栈中被管理，这个栈也被称为一个Task。

一个Task中的Activity可以来自不同的App，同一个App的Activity也可能不在一个Task中。

任务栈分为前台任务栈和后台任务栈。

#### 2.1 TaskRecord

* ActivityRecord用来记录一个Activity的所有信息。
* TaskRecord中包含一个或多个ActivityReord。
* TaskRecord用来表示Activity的任务栈，用来管理栈中的ActivityRecord。
* ActivityStack又包含了一个或多个TaskRecord，它是TaskRecord的管理者。

**表**：TaskRecord的部分重要成员变量。如下：

| 名称        | 类型                       | 说明                           |
| ----------- | -------------------------- | ------------------------------ |
| taskId      | int                        | 任务栈的唯一标识符             |
| affinity    | String                     | 任务栈的倾向性                 |
| intent      | Intent                     | 启动这个任务栈的Intent         |
| mActivities | ArrayList< ActivityRecord> | 按照历史顺序排列的Activity记录 |
| mStack      | ActivityStack              | 当前归属的ActivityStack        |
| mService    | ActivityManagerService     | AMS引用                        |

#### 2.2 TaskAffinity

TaskAffinity，可以翻译为任务相关性。用来指定Activity希望归属的栈。默认情况下，同一个应用程序所有的Activity都有着相同的`taskAffinity`，也就是应用的包名。

如何指定Activity的TaskAffinity？在AndroidManifest中设置Activity的`android:taskAffinity`属性：

```xml
<activity android:name=".MainActivity" 
          android:launchMode="singleTask"
          android:taskAffinity="com.nj.task1"/>
```

`taskAffinity`在下面两种情况时会产生效果。

* (1) `taskAffinity`与`FLAG_ACTIVITY_NEW_TASK`或者`singleTask`配合。如果新启动Activity的`taskAffinity`和栈`taskAffinity`相同则加入到该栈中；如果不同，就会创建新栈。
* (2) `taskAffinity`与`allowTaskReparenting`配合。如果`allowTaskReparenting`为true，说明Activity具有转移的能力。拿之前的发邮件为例，当社交应用启动了发送邮件的Activity，此时发送邮件的Activity是和社交应用处于同一个栈中的，并且这个栈位于前台。如果发送邮件的Activity的`allowTaskReparenting`设置为true，此后Email应用所在的栈位于前台时，发送邮件的Activity就会由社交应用的栈中转移到与它更亲近的邮件应用（taskAffinity相同）所在的栈中。

#### 2.3 adb查看任务栈

想要查看当前系统的任务栈，在命令行执行如下命令：

```
adb shell dumpsys activity
```

执行上面的命令，会打印出很多有关Activity的消息，其中会有如下几行内容：

```java
ACTIVITY MANAGER RECENT TASKS (dumpsys activity recents)
mRecentsUid=10038
mRecentsComponent=ComponentInfo{com.google.android.apps.nexuslauncher/com.android.quickstep.RecentsActivity}
  Recent tasks:
  * Recent #0: TaskRecord{ca18582 #170 A=com.nj.testappli U=0 StackId=166 sz=1}
  * Recent #1: TaskRecord{8f277ab #5   
    I=com.google.android.apps.nexuslauncher/.NexusLauncherActivity U=0 StackId=0 sz=1}
  * Recent #2: TaskRecord{2d3345a #3 I=com.android.settings/.FallbackHome U=0 StackId=-1 
    sz=0}
```

上面的内容，就是当前系统的存在任务栈，我们可以看到有3个任务栈，`#0`是栈顶的元素，也就是当前正在运行的应用。其中TaskRecord就代表一个任务栈对象。`#170`是任务栈的id，`A=com.nj.testappli`是任务栈的taskAffinity属性，`StackId`是任务栈所在的ActivityStack的id，sz是当前任务栈中的Activity个数，也就是对应的ActivityRecord的个数。调用`adb shell dumpsys activity recents`可以查看更为详细的任务栈信息。

如果我们想要查看当前有哪些任务栈，并且每个任务栈中有哪些Activity，可以执行如下面命令：

```
adb shell dumpsys activity activities
```

运行命令后的结果大致如下：

```
ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)
  Stack #169: type=standard mode=fullscreen
    ... // 省略Stack的一些信息
    * TaskRecord{6fc9357 #173 A=com.nj.testappli U=0 StackId=169 sz=3}
      ... // 省略TaskRecord的一些信息 
      * Hist #2: ActivityRecord{1d0346f u0 com.nj.testappli/.ThridActivity t173}
          packageName=com.nj.testappli processName=com.nj.testappli
          ... // 省略ActivityRecord的一些信息
      * Hist #1: ActivityRecord{48b8cbc u0 com.nj.testappli/.SecondActivity t173}
          packageName=com.nj.testappli processName=com.nj.testappli
      * Hist #0: ActivityRecord{964079c u0 com.nj.testappli/.MainActivity t173}
          packageName=com.nj.testappli processName=com.nj.testappli
          
    Running activities (most recent first):
      TaskRecord{6fc9357 #173 A=com.nj.testappli U=0 StackId=169 sz=3}
        Run #2: ActivityRecord{1d0346f u0 com.nj.testappli/.ThridActivity t173}
        Run #1: ActivityRecord{48b8cbc u0 com.nj.testappli/.SecondActivity t173}
        Run #0: ActivityRecord{964079c u0 com.nj.testappli/.MainActivity t173}

    mResumedActivity: ActivityRecord{1d0346f u0 com.nj.testappli/.ThridActivity t173}
    mLastPausedActivity: ActivityRecord{48b8cbc u0 com.nj.testappli/.SecondActivity t173}

  Stack #0: type=home mode=fullscreen
    * TaskRecord{8f277ab #5 I=com.google.android.apps.nexuslauncher/.NexusLauncherActivity 
      U=0 StackId=0 sz=1}
      * Hist #0: ActivityRecord{70c4b5e u0com.google.android.apps.nexuslauncher/ 	
        .NexusLauncherActivity t5}
    Running activities (most recent first):
      TaskRecord{8f277ab #5 I=com.google.android.apps.nexuslauncher/.NexusLauncherActivity 	
      U=0 StackId=0 sz=1}
        Run #0: ActivityRecord{70c4b5e u0 
        com.google.android.apps.nexuslauncher/.NexusLauncherActivity t5}

    mLastPausedActivity: ActivityRecord{70c4b5e u0 
    com.google.android.apps.nexuslauncher/.NexusLauncherActivity t5}
```

从上面可以看到，ActivityStack包含一个或者多个TaskRecord，一个TaskRecord包含一个或者多个ActivityRecord，一个ActivityRecord就对应一个Activity实例。

在Running aitivities部分的信息，可以看到一个TaskRecord中的具体Activity调用栈的顺序，在之后的mResumedActivity就代表正在运行的Activity，mLastPausedActivity代表上一个调用onPause的Activity。

#### 2.4 案例分析

假设，现在有两个应用：

* A应用，包名：com.nj.appA。A应用中包含几个Activity，分别是A1（应用主入口）、A2、A3。
* B应用，包名：com.nj.appB。B应用中包含几个Activity，分别是B1（应用主入口）、B2、B3。

接下来分几种情况来分析调用栈的使用和创建情况：

1. A1启动A2，A2是standard模式。

   A2会添加在A1所在的任务栈中，因为它们默认的TaskAffinity都为A的包名`com.nj.appA`。

2. A1启动A2，并且设置`FLAG_ACTIVITY_NEW_TASK`，A2是singleTask。

   A2会添加在A1所在的任务栈中，虽然设置了`FLAG_ACTIVITY_NEW_TASK`，但是它们的TaskAffinity相等，并且该任务栈已经存在，所以不会创建新的栈。

3. A1启动A2，A2设置了`android:taskAffinity="com.nj.task1"`，A2的启动模式为standard。

   A2会添加到A1所在的任务栈中，虽然设置了不同的TaskAffinity，但是启动模式为standard，不会创建新的栈。A2启动模式设置为singleTop同理。

4. A1启动A2，并且设置`FLAG_ACTIVITY_NEW_TASK`，A2设置了`android:taskAffinity="com.nj.task1"`，A2的启动模式为standrad。

   此时会创建一个新的栈，A2将添加到新的栈中。这就是之前讲到的TaskAffinity需要和`FLAG_ACTIVITY_NEW_TASK`配合使用。

5. A1启动A2，A2设置了`android:taskAffinity="com.nj.task1"`，A2的启动模式为singleTask。

   此时会创建一个新的栈，A2将添加到新的栈中。这就是之前讲到的TaskAffinity需要和singltTask配合使用。

6. A1启动B1，B1是standard模式。

   B1会添加到A1所在的任务栈中。这也就证明了一个Task中的Activity可以来自不同的App。

7. A1启动B1，并且设置`FLAG_ACTIVITY_NEW_TASK`，B1是standard模式。

   此时会创建一个新的栈，新的栈的TaskAffinity是B1应用的包名`com.nj.appB`。B1将添加到新的栈中。

8. A1启动B1，B1是singleTask模式

   此时会创建一个新的栈，新的栈的TaskAffinity是B1应用的包名`com.nj.appB`。B1将添加到新的栈中。这个结果的原理其实和4、5的结果是同理的，B1是应用B的Activity，默认的TaskAffinity也是B应用的包名，所以等同于设置了TaskAffinity，所以和4、5的结果一致。

9. A1启动B1，B1是singleTask模式，B1设置了`android:taskAffinity="com.nj.appA"`，也就是将B1设置成和应用A包名一样的TaskAffinity。

   此时B1会添加到A1所在的栈中， 并不会创建新的栈。因为B1要求的TaskAffinity为`com.nj.appA`的栈已经存在了，所以不会创建新的栈。

10. A1启动B1，并且设置`FLAG_ACTIVITY_NEW_TASK`，B1设置了`android:taskAffinity="com.nj.appA"`。

    结果和9一致。

11. A1启动B2，B2模式为standard，B2设置了`allowTaskReparenting`属性为true，设置了`MAIN`入口（否则无法隐式启动），具体设置如下：

    ```xml
    <activity android:name=".BMainActivity" android:allowTaskReparenting="true">
        <intent-filter>
            <action android:name="android.intent.action.MAIN"/>
        </intent-filter>
    </activity>
    ```

    此时B2会添加到A1所在的栈，然后点击Home键回到桌面，再点击B应用的图标显示的会是B2，按下back键，显示B1。说明B2从A1所在的栈，移动到了B1所在的栈。因为A1启动B2的时候，B2倾向的栈是TaskAffinity为B的包名`com.nj.appB`的栈，但是这个栈不存在，各种设置也没有要求创建新的栈，所以只能添加到A1所在的栈。当我们点击B应用的图标是，B应用的栈创建，B1添加到栈中，此时B2发现我想要的栈创建了，所以就会移动到它倾向的栈中。（书本上的内容，实际Android 28 测试并未达到想要的效果）

