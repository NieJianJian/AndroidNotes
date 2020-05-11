## AIDL

Messenger是串行处理消息，如果有大量消息发送到服务端，Messenger就不合适了。

AIDL可以实现跨进程的方法调用，Messenger无法做到。

### 1. AIDL使用流程

1. 服务端
   * 首先要创建一个Service用来监听客户端的连接请求；
   * 然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明；
   * 最后在Service中实现这个AIDL接口即可。

2. 客户端
   * 首先需要绑定服务端的Service；
   * 绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。

***

### 2. AIDL支持的数据类型

* 基本数据类型（int、long、char、boolean、double等）；
* String和CharSequence；
* List：只支持ArrayList，里面每个元素都必须能被AIDL支持；
* Map：只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value；
* Parcelable：所有实现了Parcelable接口的对象；
* AIDL：所有的AIDL接口本身也可以在AIDL文件中使用。

***

### 3. AIDL使用注意事项

* 自定义的Parcelable对象和AIDL对象必须要显式import进来，不管它们是否和当前的AIDL文件位于同一个包内。
* 如果AIDL文件中使用到了自定义的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并且其中声明它为Parcelable类型。
* AIDL中除了基本数据类型，其他类型的参数必须标上方向：in、out、inout。in表示输入型参数，out表示输出型参数，inout表示输入输出型参数。
* AIDL接口中只支持方法，不支持声明静态常量，这一点区别于传统的接口。
* AIDL的包结构在服务端和客户端要保持一致，否则运行会出错。因为客户端需要反序列化服务端中和AIDL接口相关的所有类，如果类的完整路径不一样的话，就无法成功反序列化，程序无法正常运行。（建议把所有和AIDL相关的类和文件放入同一个包中，当客户端是另外一个应用时，可以直接把整个包复制到客户端工程中）。

***

### 4. AIDL的使用过程

新建3个文件Book.java、Book.aidl、IBookManager.aidl，原有包名路径（com.nj.testappli）。

1. 新建Java包com.nj.testappli.aidl，然后创建图书信息Book.java类，实现Parcelable接口，代码如下：	

   ```java
   package com.nj.testappli.aidl;
   import android.os.Parcel;
   import android.os.Parcelable;
   public class Book implements Parcelable {
       public int bookId;
       public String bookName;
       public Book(int id, String name) {
           this.bookId = id;
           this.bookName = name;
       }
       @Override
       public int describeContents() {
           return 0;
       }
       @Override
       public void writeToParcel(Parcel dest, int flags) {
           dest.writeInt(bookId);
           dest.writeString(bookName);
       }
       public static final Creator<Book> CREATOR = new Creator<Book>() {
           @Override
           public Book createFromParcel(Parcel in) {
               return new Book(in);
           }
           @Override
           public Book[] newArray(int size) {
               return new Book[size];
           }
       };
       protected Book(Parcel in) {
           bookId = in.readInt();
           bookName = in.readString();
       }
   }
   ```

2. 在包com.nj.testappli.aidl下，按照`右键->New->AIDL->AIDL File`的步骤，创建Book.aidl文件。

   **注意**：由于存在了Book.java，所以创建Book.adil文件时不允许的，因为同名文件，所以先创建一个别的名称，创建完成后，再改成Book.adil就可以。

   点击创建后，会在src/main目录下，生成一个和java目录同级的目录，名字叫aidl，然后内部的包结构和java下面的结构一致，新创建的Book.aidl文件也会创建在这里。

   新创建的Book.aidl文件内容如下：

   ```java
   // Book.aidl
   package com.nj.testappli.aidl;
   
   parcelable Book;
   ```

   Book.aidl是Book类在AIDL中的声明，注意parcelable是小写。

3. 在Book.adil所在的目录下继续创建IBookManager.aidl文件，里面声明两个方法，内容如下：

   ```java
   // IBookManager.aidl
   package com.nj.testappli.aidl;
   
   import com.nj.testappli.aidl.Book;
   interface IBookManager {
       List<Book> getBookList();
       void addBook(in Book book);
   }
   ```

   **注意**：尽管Book和IBookManager在同一个包下，但是还是需要手动导入。

4. 执行make project或者rebuild Project命令，生成IBookManager.aidl的Binder类，结果会在build/generated/aidl_source_output_dir里面生成IBookManager.java类。代码如下：

   ```java
   /*
    * This file is auto-generated.  DO NOT MODIFY.
    * Original file: /Users/niejianjian/Work/Source/as30/TestAppli/app/src/main/aidl/com/nj/testappli/aidl/IBookManager.aidl
    */
   package com.nj.testappli.aidl;
   
   public interface IBookManager extends android.os.IInterface {
       /* Local-side IPC implementation stub class. */
       public static abstract class Stub extends android.os.Binder implements
               com.nj.testappli.aidl.IBookManager {
           private static final java.lang.String DESCRIPTOR = "com.nj.testappli.aidl.IBookManager";
   
           /* Construct the stub at attach it to the interface. */
           public Stub() {
               this.attachInterface(this, DESCRIPTOR);
           }
   
           /* Cast an IBinder object into an com.nj.testappli.aidl.IBookManager interface,
            * generating a proxy if needed. */
           public static com.nj.testappli.aidl.IBookManager asInterface(android.os.IBinder obj) {
               if ((obj == null)) {
                   return null;
               }
   
               android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
               if (((iin != null) && (iin instanceof com.nj.testappli.aidl.IBookManager))) {
                   return ((com.nj.testappli.aidl.IBookManager) iin);
               }
               return new com.nj.testappli.aidl.IBookManager.Stub.Proxy(obj);
           }
   
           @Override
           public android.os.IBinder asBinder() {
               return this;
           }
   
           @Override
           public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
               java.lang.String descriptor = DESCRIPTOR;
               switch (code) {
                   case INTERFACE_TRANSACTION: {
                       reply.writeString(descriptor);
                       return true;
                   }
                   case TRANSACTION_getBookList: {
                       data.enforceInterface(descriptor);
                       java.util.List<com.nj.testappli.aidl.Book> _result = this.getBookList();
                       reply.writeNoException();
                       reply.writeTypedList(_result);
                       return true;
                   }
                   case TRANSACTION_addBook: {
                       data.enforceInterface(descriptor);
                       com.nj.testappli.aidl.Book _arg0;
                       if ((0 != data.readInt())) {
                           _arg0 = com.nj.testappli.aidl.Book.CREATOR.createFromParcel(data);
                       } else {
                           _arg0 = null;
                       }
                       this.addBook(_arg0);
                       reply.writeNoException();
                       return true;
                   }
                   default: {
                       return super.onTransact(code, data, reply, flags);
                   }
               }
           }
   
           private static class Proxy implements com.nj.testappli.aidl.IBookManager {
               private android.os.IBinder mRemote;
   
               Proxy(android.os.IBinder remote) {
                   mRemote = remote;
               }
   
               @Override
               public android.os.IBinder asBinder() {
                   return mRemote;
               }
   
               public java.lang.String getInterfaceDescriptor() {
                   return DESCRIPTOR;
               }
   
               @Override
               public java.util.List<com.nj.testappli.aidl.Book> getBookList() throws android.os.RemoteException {
                   android.os.Parcel _data = android.os.Parcel.obtain();
                   android.os.Parcel _reply = android.os.Parcel.obtain();
                   java.util.List<com.nj.testappli.aidl.Book> _result;
                   try {
                       _data.writeInterfaceToken(DESCRIPTOR);
                       mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                       _reply.readException();
                       _result = _reply.createTypedArrayList(com.nj.testappli.aidl.Book.CREATOR);
                   } finally {
                       _reply.recycle();
                       _data.recycle();
                   }
                   return _result;
               }
   
               @Override
               public void addBook(com.nj.testappli.aidl.Book book) throws android.os.RemoteException {
                   android.os.Parcel _data = android.os.Parcel.obtain();
                   android.os.Parcel _reply = android.os.Parcel.obtain();
                   try {
                       _data.writeInterfaceToken(DESCRIPTOR);
                       if ((book != null)) {
                           _data.writeInt(1);
                           book.writeToParcel(_data, 0);
                       } else {
                           _data.writeInt(0);
                       }
                       mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                       _reply.readException();
                   } finally {
                       _reply.recycle();
                       _data.recycle();
                   }
               }
           }
   
           static final int TRANSACTION_getBookList =
                   (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
           static final int TRANSACTION_addBook =
                   (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
       }
   
       public java.util.List<com.nj.testappli.aidl.Book> getBookList()
               throws android.os.RemoteException;
   
       public void addBook(com.nj.testappli.aidl.Book book)
               throws android.os.RemoteException;
   }
   ```

   IBookManager.java这个类，继承了IInterface接口，同时它自己也是个接口。**所有可以在Binder中传输的接口都需要继承IInterface接口**。

   我们来分析以下IBookManager.java的逻辑：

   * 首先，它声明了getBookList和addBook两个方法。这是在IBookManager.adil中所声明的方法，
   * 声明了两个整型的id分别用于标识上述两个方法。这两个id用于在transact过程中区分客户端所请求的到底是哪个方法。
   * 声明一个内部类Stub，这个Stub是一个Binder类。
   * 当客户端和服务端位于同一个进程的时候，方法调用不会走进程的transact过程；当两个位于不同进程时，方法调用需要走transact过程。这个逻辑由Stub的内部代理类Proxy来完成。

   这个接口的核心是它的内部类Stub和Stub的内部代理类Proxy，下面介绍这两个类中方法的含义：

   * **DESCRIPTOR**

     Binder唯一标识，一般用当前Binder类名表示，比如本例中的"com.nj.testappli.aidl.IBookManager"。

   * **asInterface**(android.os.IBinder obj)

     用于将服务端的Binder对象转换程客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一个进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.Proxy对象。

   * **asBinder**

     此方法用于返回当前的Binder对象。

   * **onTransact**

     这个方法运行在服务端的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法处理。

     * 服务端通过code可以确定客户端所请求的目标方法；
     * 接着从data中取出目标方法所需要的参数（如果目标方法有参数的话）；
     * 然后执行目标方法；
     * 目标方法执行完毕后，向reply中写入返回值（如果有返回值的话）。

     **注意**：如果此方法返回false，那么客户端请求会失败，所以可以利用这个特性做权限校验。

   * **Proxy#getBookList**

     这个方法运行在客户端，当客户端调用此方法时，它的内部实现是这样的：

     * 首先创建该方法所需的输入型Parcel对象`_data`、输出型Parcel对象`_reply`和返回值对象List；
     * 然后把该方法的参数信息写入到`_data`中（如果有参数）；
     * 接着调用transact方法来发起RPC（远程过程调用）请求，同时当前线程挂起；
     * 然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行，并从`_reply`中取出RPC过程的返回结果；
     * 最后返回`_reply`中的数据。

   * **Proxy#addBook**

     这个方法运行在客户端，执行过程和getBookList一样，addBook没有返回值，所以不需要从`_reply`中取出返回值。

5. 远程服务端Service的实现

   创建一个Service，代码如下：

   ```java
   public class BookManagerService extends Service {
       private static final String TAG = "BMS";
       private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();
       private Binder mBinder = new IBookManager.Stub() {
           @Override
           public List<Book> getBookList() throws RemoteException {
               return mBookList;
           }
           @Override
           public void addBook(Book book) throws RemoteException {
               mBookList.add(book);
           }
       };
       @Override
       public void onCreate() {
           super.onCreate();
           mBookList.add(new Book(1, "Android));
           mBookList.add(new Book(2, "ios"));
       }
       @Override
       public IBinder onBind(Intent intent) {
           return mBinder;
       }
   }
   ```

   这里使用了支持并发读写的CopyOnWriteArrayList。因为AIDL方法是在线程池中执行的，因此多个客户端同时连接的时候，会存在多个线程访问的情形，所以需要处理同步问题。

   **Q**：AIDL中能够使用的List只有ArrayList，为什么这里却使用CopyOnWriteArrayList（并非继承ArrayList）？

   **A**：因为AIDL支持的是抽象的List，而List只是一个接口，因此虽然服务端返回的是CopyOnWriteArrayList，

   但是在Binder中会按照List的规范去访问数据并最终形成一个新的ArrayList传递给客户端，所以可行。类似的还有ConcurrentHashMap。

6. 清单文件中注册该Service，指定独立进程

   ```xml
   <service android:name=".aidl.BookManagerService" android:process=":bookmanager"/>
   ```

7. 客户端的实现

   * 首先绑定远程服务；
   * 绑定成功后将服务端返回的Binder对象转换成AIDL接口；
   * 然后通过该接口去调用服务端的远程方法。

   ```java
   public class MainActivity extends Activity {
       private ServiceConnection mConnection = new ServiceConnection() {
           @Override
           public void onServiceConnected(ComponentName name, IBinder service) {
               IBookManager bookManager = IBookManager.Stub.asInterface(service);
               try {
                   List<Book> list = bookManager.getBookList();
                   Log.i("niejianjian", " query book list, list type "
                           + list.getClass().getCanonicalName());
                   Log.i("niejianjian", " query book list: " + list.toString());
               } catch (Exception e) {
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
           Intent intent = new Intent(this, BookManagerService.class);
           bindService(intent, mConnection, BIND_AUTO_CREATE);
       }
       @Override
       protected void onDestroy() {
           super.onDestroy();
           unbindService(mConnection);
       }
   }
   ```

   调用getBookList，服务端的方法可能需要很久才能执行完毕，这个时候可能会导致ANR。

   运行程序，打印的log如下：

   ```
   I/niejianjian:  query book list, list type java.util.ArrayList
   I/niejianjian:  query book list: [[bookId:1,bookName:Android], bookId:2,bookName:ios]
   ```

8. 如果想要有新书时服务端主动告诉客户端，也就是观察者模式。可以采用注册回调listener的方式。

9. 最后退出程序的时候，要在onDestroy中解注册回调listener，但是会失效。

   **Q**：为什么解注册会失效？

   **A**：虽然我们在注册和解注册的过程中使用同一个客户端对象，但是通过Binder传递到服务端后会产生两个全新的对象。因为对象不能跨进程传输，对象的跨进程传输本质上都是反序列化的过程，这就是为什么AIDL中自定义的对象要实现Parcelable接口了。

   **Q**：如何才能实现解注册？

   **A**：使用RemoteCallbackList，这是系统专门提供的用于删除跨进程listener的接口。

   RemoteCallbackList内部是一个Map接口，专门用来保存AIDL的回调，Map的key是IBinder类型，value时Callback类型。当客户端解注册的时候，我们只要遍历服务端所有的listener，找到那个和解注册listener具有相同Binder对象的服务端listener并把它删除即可。

***

### 5. AIDL权限验证

默认情况下，我们的远程服务任何人都可以连接，这不是我们愿意看到的。所以我们必须给服务加入权限验证功能，权限验证失败则无法访问。AIDL权限验证有两种常用方法。

* 方法一

  在onBind中进行验证，验证不通过直接返回null，这样验证失败的客户端直接无法绑定服务，验证方式有多种，比如使用permission验证，首先在AndroidManifest中声明所需的权限：

  ```java
  <permission
      android:name="com.ni.testappli.permission.ACCESS_BOOK_SERVICE"
      android:protectionLevel="normal"/>
  ```

  定义权限后，在BookManagerService的onBind方法中使用：

  ```java
  public IBinder onBind(Intent intent) {
      int check = checkCallingOrSelfPermission(
              "com.ni.testappli.permission.ACCESS_BOOK_SERVICE");
      if (check == PackageManager.PERMISSION_DENIED) {
          return null;
      }
      return mBinder;
  }
  ```

  如果我们自己内部的应用想要绑定服务，只需要AndroidManifest中声明permission即可：

  ```java
  <uses-permission android:name="com.ni.testappli.permission.ACCESS_BOOK_SERVICE"/>
  ```

* 方法二：

  可以在服务端的onTransact方法中进行权限验证，如果验证失败则直接返回false，具体验证方式很多，可以采用方法一中的permission验证，也可以验证Uid和Pid，通过getCallingUid和getCallingPid可以拿到客户端所属的应用的Uid和Pid，通过这两个参数我们可以做一些验证，比如验证包名。

***

### 6. 其他事项

* 客户端调用远程服务的方法，被调用的方法运行在服务端的Binder线程池中，同时客户端线程会被挂起，这个时候如果服务端方法执行比较耗时，就会导致客户端线程长时间地阻塞，如果时UI线程，就会ANR。
* 由于服务端的方法本身就运行在服务端的Binder线程池中，所以服务端本身可以执行大量耗时操作，这个时候切记不要在服务端方法中开线程去进行异步任务。
* 当远程服务端需要调用客户端的listener的方法时，被调用的方法也运行在Binder线程池，只不过是客户端的线程池。此时想要访问UI，请使用Handler切换到UI线程。
* BInder是可能意外死亡的，往往由于服务端进程的意外停止，这时我们需要重新连接服务。
  * 方法一：给Binder设置DeathRecipient监听，当Binder死亡时，我们会收到binderDied方法的回调，在binderDied方法中我们可以重连远程服务。
  * 方法二：在onServiceDisconnected中重连远程服务。

