## IPC机制

### 目录

* [一. IPC介绍](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#一-ipc介绍)
* [二. 基础概念介绍](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#二-基础概念介绍)
  * [1. Serializable接口](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#1-serializable接口)
  * [2. Parcelable接口](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#2-parcelable接口)
  * [3. Binder](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#3-binder)
* [三. IPC的实现方式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#三-ipc的实现方式)
  * [1. 使用Bundle](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#1-使用bundle)
  * [2. 使用文件共享](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#2-使用文件共享)
  * [3. 使用Messenger](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#3-使用messenger)
  * [4. AIDL](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#4-aidl)
  * [5. ContentProvider](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#5-contentprovider)
  * [6. Socket](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/IPC.md#6-socket)

***

***

***

### 一. IPC介绍

IPC是Inter-Process Communication的缩写，也就是进程间通信的意思。

* 为什么会有进程间通信的问题？

  * 应用之间数据传递
  * 单个应用多进程模式下需要数据传递。

* 使用多进程的方法。

  只有一种方法，给四大组件在AndroidManifest中指定`android:process`属性。

  ```xml
  <activity android:name="AActivity"/>
  <activity android:name="BActivity" android:process="com.nj.test.b"/>
  <activity android:name="CActivity" android:process=":c"/>
  ```

  假设应用的包名为`com.nj.test`，则应用的默认进程名称为`com.nj.test`。

  * AActivity启动后在默认进程`com.nj.test`中。

  * BActivity启动后在指定进程`com.nj.test.b`中。

  * CActivity启动后在指定进程`com.nj.test:c`中。":"的含义是在指定进程名前面加上包名。

    * ":"开头的进程属于应用的私有进程，其他应用的组件不可以和它跑在一个进程中。
    * 不以":"开头的进程属于全局进程，可以通过ShareUID方式和它跑在一个进程。

    > **uid**：Android系统为每个应用分配唯一的UID，具有相同UID的应用才能共享数据。
    >
    > * 两个应用通过shareUID共享数据是有要求的，必须uid相同并且签名相同。

* 多进程运行机制

  ```java
  public class Contants {
      public static int test = 1;
  }
  ```

  AActivity中把变量修改为2，再启动BActivity调用`test`的值，发现值是1，并不是修改后的2。

  * Android中每一个进程都是一个单独的JVM，也就是有了不同的存储空间，所以不同的虚拟机访问同一个类的对象会产生多个副本。（两个进程中都存在一个Contants类）

  一般来说，**多进程会造成以下问题**：

  * 静态成员和单例模式会完全失效。

  * 线程同步机制完全失效。

    同第一个问题本质一样，不同进程锁的不是一个对象。

  * SharedPreferences的可靠性降低。

    不支持两个进程同时执行读写，会导致并发问题，和两个线程操作一个变量的读写类似。

  * Application会多次创建。

    不同进程会分配不同的虚拟机，等于是启动了一个新的应用进程。

***

***

***

### 二. 基础概念介绍

**序列化**：是将对象状态转换为可保持或传输的格式的过程。与序列化相对的是反序列化，它将流转换为对象。这两个过程结合起来，可以轻松地存储和传输数据。

> 把对象转换为字节序列的过程称为对象的序列化。
>
> 把字节序列恢复为对象的过程称为对象的反序列化。

**为什么要序列化**？序列化可以保存对象的属性到本地文件、数据库、网络流、rmi以方便数据传输，当然这种传输可以是程序内的也可以是两个程序间的

#### 1. Serializable接口

* **概念**：Java提供的序列化接口，一个类实现了Serializable接口，它的对象才能序列化。
* **使用方法**：实现Serializable接口，并且在类中声明`serializableUID`变量。

下面看一个例子：

```java
public class User implements Serializable {
    private static final long serialVersionUID = -5596372558592625585L;
    public int userId;
    public String userName;
    ...
}
```

使用Serializable方式来实现对象的序列化非常简单，基本所有工作都被系统完成了。

进行序列化只需要采用ObjectOutputStream和ObjectInputStream即可，如下：

```java
String path = Environment.getExternalStorageDirectory().getPath();
File file = new File(path, "cache.txt");

// 序列化过程
User user = new User(20, "nie");
ObjectOutputStream stream = new ObjectOutputStream(
  new FileOutputStream(file.getPath())); // 没有实现Serializable接口会报错
stream.writeObject(user);
stream.close();

// 反序列化过程
ObjectInputStream in = new ObjectInputStream(new FileInputStream(file.getPath()));
User o = (User) in.readObject();
in.close();
```

* `serializableUID`的工作机制：序列化的时候系统会把当前类的serializableUID写入序列化文件中，反序列化的时候系统给你会检测文件中的serializableUID。看是否和当前类的serializableUID一致。如果一致说明序列化类的版本和当前类的版本是相同的，否则说明当前类发生了改变。

* 需要注意的点：

  * 静态成员变量属于类不属于对象，所以不会参与序列化过程
  * 使用`transient`关键字标记的成员变量不参与序列化过程。

* 如果修改系统默认的序列化过程，重写如下两个方法：

  ```java
  private void writeObject(java.io.ObjectOutputStream out)
          throws IOException {
  }
  private void readObject(java.io.ObjectInputStream in)
          throws IOException, ClassNotFoundException {
  }
  ```

***

#### 2. Parcelable接口

　　Parcelable设计的初衷是为了解决Serializable效率慢的问题，为了在程序内不同组件间以及不同Android进程间(AIDL)高效的传输数据而设计。这些数据仅在内存中存在，Parcelable是通过IBinder通信的消息的载体。

　　Parcelable的性能比Serializable好，在内存开销方面较小，所以**在内存间数据传输时推荐使用Parcelable**，如activity间传输数据，而Serializable可将数据持久化方便保存，所以**在需要保存或网络传输数据时选择Serializable**，因为android不同版本Parcelable可能不同，所以不推荐使用Parcelable进行数据持久化。

|          | Serializable             | Parcelable                       |
| -------- | ------------------------ | -------------------------------- |
| 开销     | 开销大，需要大量I/O操作  | 开销小，内存中存在               |
| 效率     | 效率低，使用简单         | 效率高，使用麻烦                 |
| 使用场景 | 需要保存或网络传输数据等 | 内存中传输，如Activity间传递数据 |

下面看一个典型的用法：

```java
public class User implements Parcelable {
    public int userId;
    public String userName;
    public User(int id, String name) {
        this.userId = id;
        this.userName = name;
    }
    @Override
    public int describeContents() {
        return 0;
    }
    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(userId);
        dest.writeString(userName);
    }
    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }
        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
    protected User(Parcel in) {
        userId = in.readInt();
        userName = in.readString();
    }
}
```

Parcelable方法说明表如下：

| 方法                                | 功能                                                         | 标记位 |
| ----------------------------------- | ------------------------------------------------------------ | ------ |
| createFormParcel(Parcel in)         | 从序列化后的对象中创建原始对象                               |        |
| newArray(int size)                  | 创建指定长度的原始对象数组                                   |        |
| User(Parcel in)                     | 从序列化后的对象中创建原始对象                               |        |
| writeToParcel(Parcel out,int flags) | 将当前对象写入序列化结构中，其中flags标识有两种值：0或1。当为1时当前对象需要作为返回值返回，不能立即释放资源，几乎所有情况都为0。 |        |
| describeContents                    | 放回当前对象的内容描述。如果含有文件描述，返回1，否则返回0，机会所有情况都是0。 |        |

系统提供了实现Parcelable接口的类，如Intent、Bundle、Bitmap等。List和Map也能序列化，前提是他们里面的每个元素都是可序列化的。

***

#### 3. Binder

[Binder](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/Binder.md)

***

***

***

### 三. IPC的实现方式

* 使用Bundle
* 使用文件共享
* 使用Messenger
* 使用AIDL
* 使用ContentProvider
* 使用Socket

#### 1. 使用Bundle

四大组件的Activity、Service、Receiver都是支持在Intent中传递Bundle数据的。因为Bundle实现了Parcelable接口。Bundle可以传递基本类型、实现了序列化接口的对象以及一些Android支持的特殊对象，具体的可以查看Bundle类。

***

#### 2. 使用文件共享

两个进程通过读/写同一个文件来交换数据。Android系统基于Linux，使得其并发读/写文件可以没有限制地进行，甚至两个线程同时对同一个文件进行写操作都是允许的，尽管这可能出问题。

通过文件交换数据很好使用，除了可以交换一些文本信息外，我们还可以序列化一个对象到文件系统中地同时从另一个进程中恢复这个对象，比如前面序列化实现Serializable接口的对象。但是反序列化得到的对象只是内容一样，但实际上是两个对象。

**缺点**：如果并发读写可能导致数据读取的不是最新的数据。如果并发写可能问题更严重了。

SharedPreferences也属于文件的一种，但是系统底层对它的读写有一定的**缓存**策略，即在内存中会有一份ShreadPreferences文件的缓存，因此在多进程模式下，系统对它的读写变得不可靠，当面对高并发的读写访问，ShreadPreferences有很大几率会丢失数据，因此不建议在进程间通信中使用。

***

#### 3. 使用Messenger

将需要传递到数据放入Message对象中，通过Messenger信使进行传递，进而实现进程间数据传递。

Messenger是一种轻量级的IPC，底层是AIDL，Messenger的构造函数就可以看出：

```java
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}
public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```

Messenger一次只能处理一个请求，所以在服务端不用考虑线程同步问题。

使用方式如下：

**1. 服务端进程**

* 首先，服务端创建一个Service来处理客户端的连接请求。
* 然后创建一个Handler并通过它来创建一个Messenger对象。
* 最后在Service的onBind中返回这个Messenger对象底层的Binder即可。

**2. 客户端进程**

* 首先要绑定服务端的Service。
* 绑定成功后用服务端返回的IBinder对象创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，发消息类型为Message。（个人理解服务端返回的IBinder对象有点像注入的回调对象，然后调用对象的方法就完成了回调，服务端就收到了消息）。
* 如果客户端想要得到服务端的回复，就需要像服务端一样，创建一个Handler创建一个新的Messenger。
* 然后把创建的Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端了（个人理解等于是把客户端的回调对象通过Message的replayTo传递到服务端，服务端调用对象的方法就完成了回调，客户端就收到了消息）。

来看一个简单的例子，这个例子暂时服务端无法回应客户端。

```java
public class MessengerService extends Service {
    // MessengerHandler用来处理客户端发来的消息
    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 1:
                    Log.i("niejianjian", "receive msg from Client:"
                            + msg.getData().getString("msg"));
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }
    // 将Messenger和Handler进行关联，作用是将客户端发送来的消息传递给Handler处理
    private final Messenger mMessenger = new Messenger(new MessengerHandler());
    @Override
    public IBinder onBind(Intent intent) {
        // 返回Messenger中的Binder对象。
        return mMessenger.getBinder();
    }
}
```

```xml
<service android:name=".MessengerService" android:process=":messenger"/>
```

```java
public class MessengerActivity extends Activity {
    private Messenger mService;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 提取服务端传来的Messenger对象。
            mService = new Messenger(service);
            Message msg = Message.obtain(null, 1);
            Bundle data = new Bundle();
            data.putString("msg", "hello, this is client");
            msg.setData(data);
            try {
                mService.send(msg);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        @Override
        public void onServiceDisconnected(ComponentName name) {
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Intent intent = new Intent(this, MessengerService.class);
        // 绑定远程服务
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```

上面例子可以看出，Messenger进行数据传递需要将数据放入Message中。Messenger和Message都实现了Parcelable接口。Message所支持的类型就是Messenger所支持的传输类型。

如果需要服务端回应客户端，可做以下修改，首先处理服务端，接收到消息后，进行消息回复。

```java
    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case 1:
                    Log.i("niejianjian", "receive msg from Client:"
                            + msg.getData().getString("msg"));
                    Messenger client = msg.replyTo;
                    Message replayMessage = Message.obtain(null, 2);
                    Bundle bundle = new Bundle();
                    bundle.putString("service", "消息收到");
                    replayMessage.setData(bundle);
                    try {
                        client.send(replayMessage);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }
```

客户端也需要准备一个接收消息的Messenger和Handler，如下：

```java
private static class MessengerHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case 2:
                Log.i("niejianjian", "receive msg from Service:"
                        + msg.getData().getString("service"));
                break;
        }
    }
}
private Messenger mGetServiceMessenger = new Messenger(new MessengerHandler());
```

客户端还需要在发送消息的时候，需要把接收服务端回复的Messenger通过Message的replyTo参数传递给服务端，如下所示：

```java
mService = new Messenger(service);
Message msg = Message.obtain(null, 1);
Bundle data = new Bundle();
data.putString("msg", "hello, this is client");
msg.setData(data);
msg.replyTo = mGetServiceMessenger;
```

这样整体的功能已经完成了。

***

#### 4. AIDL

[AIDL](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/AIDL.md)

***

#### 5. ContentProvider

ContentProvider是Android提供的专门用来不用应用间进行数据共享的方式。和Messenger一样，底层实现同样也是Binder。

系统预置了很多ContentProvider，比如通讯录信息、日程表信息等，要跨进程访问这些信息，只需要通过ContentProvider的query、update、insert和delete方法即可。

**ContentProvider的结构**主要以表格的形式来组织数据，和数据库很类似。ContentProvider还支持文件数据，比如图片、视频等，在处理这类数据时可以在ContentProvider中返回文件的句柄进行操作。

**如何使用ContentProvider**?

1. 创建一个自定义的类，继承ContentProvider；

2. 实现ContentProvider的六个抽象方法：onCreate、query、update、insert、delete和getType。

   * onCreate代表ContentProvider的创建。
   * getType用来返回一个Uri请求所对应MIME类型，比如图片、视频等，如果不关注，返回null或"* / *"。
   * 剩下的四个方法对应CRUD操作，即实现对数据表的增删改查。

   六个方法都运行在ContentProvider线程中，除了onCreate运行在主线程（UI）线程，其余的五个方法均由外接回调并运行在Binder线程池。

3. 清单文件注册ContentProvider

   ```xml
   <provider
       android:authorities="com.nj.testappli.book.provider"
       android:name=".provider.BookProvider"
       android:permission="com.nj.PROVIDER"
       android:process=":provider"/>
   ```

   * `android:authorities`是ContentProvider的唯一标识。外界通过该属性进行访问，所以必须是唯一的。
   * `android:permission`如果添加了该属性，外界访问必须添加"com.nj.PROVIDER"权限。
   * ContentProvider还有读写权限`android:readPermission`和`android:writePermission`，如果声明了该权限，外界调用也必须声明相应的权限。

4. 外界访问

   ```java
   Uri uri = Uri.parse("content://com.nj.testappli.book.provider"); // 唯一标识
   getContentResolver().query(uri,null,null,null,null);
   ```

   ContentProvider通过Uri来区分外界要访问的数据集合，假设有book表和user表，为了知道外界要访问哪个表，可以为它们定义单独的Uri和Uri_Code，并通过UriMatcher的`addURI`方法来将Uri和Uri_Code关联起来。

   ```java
   public static final String AUTHORITY = "com.nj.testappli.book.provider";
   public static final int BOOK_URI_CODE = 0;
   public static final int USER_URI_CODE = 1;
   private static final UriMatcher um = new UriMatcher(UriMatcher.NO_MATCH);
   static{
       um.addURI(AUTHORITY, "book", BOOK_URI_CODE);
       um.addURI(AUTHORITY, "user", USER_URI_CODE);
   }
   ```

   外界要访问表，可以根据Uri取出的URI_CODE进行查询

   ```java
   switch() {
       case BOOK_URI_CODE:
           ...
       case USER_URI_CODE:
           ...
   }
   ```

   **注意**：query、update、insert、delete四大方法是存在多线程并发访问的，因此方法内部要做好线程同步。

***

#### 6. Socket

Socket也称为"套接字"，是网络通信中的概念。分为流式套接字和用户数据报套接字两种。

* **TCP**协议是面向连接的协议，提供稳定的双向通信功能，需要三次握手，提供超时重传机制，稳定性高，效率慢。
* **UDP**是无连接的，提供不稳定的单向通信功能，也能实现双向通信，稳定性差，效率高，无法保证数据一定能够正确传输。

Socket本身支持传输任意字节流。

#### 

