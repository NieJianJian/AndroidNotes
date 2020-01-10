## 第12章 理解ClassLoader

Java和Android中的ClassLoader的区别：

* DVM和ART加载是的dex文件，JVM加载的是Class文件。
* Java的引导类加载器是由C++编写的，Android中的引导类加载器则是用Java编写的。
* Android的继承关系要比Java继承关系复杂些，提供的功能也多。
* 由于Android中加载的不再是Class文件，因此Android中没有`ExtClassLoader`和`AppClassLoader`，代替它们的是`PathClassLoader`和`DexClassLoader`。

***

### 1 Java中的ClassLoader

### 1.1 ClassLoader的类型

 Java的类加载器主要分为两种，**系统类加载器**和**自定义类加载器**。系统类加载器包括3中，分别是：

* 1.**Bootstrap ClassLoader**（引导类加载器）

  * C/C++代码实现，不能被Java代码访问

  * 不是继承自`java.lang.ClassLoader`。

  * Java虚拟机的启动是通过Bootstrap ClassLoader创建一个初始类来完成的。

  * 主要加载指定的JDK核心类库

    * $JAVA_HOME/jre/lib目录，如`java.lang.`、`java.util.`等系统类，
    * `-Xbootclasspath`参数指定的目录。

  * 如何通过代码打印Bootstrap ClassLoader所加载目录：

    ```java
    System.out.println(System.getProperty("sun.boot.class.path"));
    ```

* 2.**Ectensions ClassLoader**（拓展类加载器）

  * 在Java中的实现类为`ExtClassLoader`。主要用于加载Java的拓展类。

  * 继承自`java.lang.ClassLoader`。

  * 主要加载以下目录中的类库：

    * 加载$JAVA_HOME/jre/lib/ext目录。
    * 系统属性`java.ext.dir`所指定的目录。

  * 如果通过代码打印Extensions ClassLoader加载目录：

    ```java
    System.out.println(System.getProperty("java.ext.dirs"));
    ```

* 3.**Application ClassLoader**（应用程序类加载器）

  * 在Java中的实现类为`AppClassLoader`，也可以称作System ClassLoader（系统类加载器），因为可以通过ClassLoader的`getSystemClassLader`方法获取到。
  * 继承自`java.lang.ClassLoader`。
  * 主要加载以下目录中的类库：
    * 当前程序的Classpath目录。
    * 系统属性java.class.path指定的目录。

* **Custom ClassLoader**（自定义类加载器）
  * 自定义类加载通过继承`java.lang.ClassLoader`类的方式来实现自己的类加载器。

### 1.2 ClassLoader的继承关系

运行一个Java程序需要用到几种类型的类加载器呢？

```java
public class ClassLoaderTest {
    public static void main() {
        ClassLoader loader = ClassLoaderTest.class.getClassLoader();
        while (loader != null) {
            System.out.println(loader);
            loader = loader.getParent();
        }
    }
}
```

打印结果为：

```
sun.misc.Launcher$AppClassLoader@75b84c92
sun.misc.Launcher$ExtClassLoader@1b6d3586
```

* 第一行说明ClassLoaderTest的类加载器是AppClassLoader。
* 第二行说明AppClassLoader的父加载器是ExtClassLoader。
* Bootstrap ClassLoader由于C/C++编写，Java中无法访问，所以没有打印出来。
* AppClassLoader的父加载器是ExtClassLoader，不代表继承自ExtClassLoader。

系统提供的类加载器有3种类型，但是提供的ClassLoader不止3个，继承关系图如下所示：

![ClassLoader的继承关系](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/classloaderextendsrelation.png)

如上图看到总共有5个ClassLoader的相关类，简单介绍如下：

* **ClassLoader**是一个抽象类，其中定义了ClassLoader的主要功能。
* **SecureClassLoader**继承了抽象类ClassLoader，但是SecureClassLoader并不是ClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。
* **URLClassLoader**继承自SecureClassLoader，可以通过URL路径从jar文件和文件夹中加载类和资源。
* **ExtClassLoader**和**AppClassLoader**都继承自URLClassLoader，他们都是Launcher的内部类，Launcher是Java虚拟机的入口应用，ExtClassLoader和AppClassLoader都是在Launcher中进行初始化的。

### 1.3 双亲委托模式

**原理**：类加载器查找Class是采用双亲委托模式。首先判断该Class是否已经加载，如果没有，则不是自身去查找而是委托给父类加载器进行查找，这样一次递归，知道委托到最顶层的Bootstrap ClassLoader。如果Bootstrap ClassLoader找到了该Class，就会直接返回，如果没有，则继续依次向下查找，如果还没找到则最后会交由自身去查找。总之，**先从下向上委托，从上向下查找**。

采取双亲委托模式的主要好处：

* 避免重复加载，如果已经加载过一次Class，就不需要再次加载，而是直接读取已经加载的Class。
* 更加安全，如果不使用双亲委托模式，就可以自定义一个String类来替代系统的String类，这显然会造成安全隐患，采用双亲委托模式会使得系统的String类在Java虚拟机启动时就被加载，也就是无法自定义String类来替代系统的String类，除非我们修改类加载器搜索类的默认算法。还有一点，只有两个类名一致且被同一个类加载器加载的类，Java虚拟机才会认为他们是同一个类。

### 1.4 自定义ClassLoader

[案例](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/Java_custom_ClassLoader.md)

***

### 2 Android中的ClassLoader

### 2.1 ClassLoader的类型

Android中的类加载器主要分为两种，**系统类加载器**和**自定义类加载器**。系统类加载器包括3中，分别是：

* 1.**BootClassLoader**

  * Android系统启动是用来预加载常用类

  * Java代码实现

    ```java
    class BootClassLoader extends ClassLoader {
        private static BootClassLoader instance;
        @FindBugsSuppressWarnings("DP_CREATE_CLASSLOADER_INSIDE_DO_PRIVILEGED")
        public static synchronized BootClassLoader getInstance() {
            if (instance == null) {
                instance = new BootClassLoader();
            }
            return instance;
        }
    }  
    ```

  * 是`ClassLoader`内部类，并继承自`ClassLoader`。

  * 是一个单例类。

  * 访问修饰符是默认的，只有同一个包中才能访问，应用程序中无法访问。

* 2.**DexClassLoader**

  * 加载dex文件以及包含dex的压缩文件（apk和jar文件）。

  * Java代码实现

    ```java
    public class DexClassLoader extends BaseDexClassLoader {
        public DexClassLoader(String dexPath, String optimizedDirectory, 
                              String librarySearchPath, ClassLoader parent) {
            super((String)null, (File)null, (String)null, (ClassLoader)null);
            throw new RuntimeException("Stub!");
        }
    }
    ```

    * **dexPath**：dex相关文件路径集合，多个路径用文件分隔符分隔，默认文件分隔符为":"；
    * **optimizedDirectory**：解压的dex文件存储路径，这个路径必须是一个内部路径，在一般情况下，使用当前应用程序的私有路径：`/data/data/<PackageNama>/...`。
    * **librarySearchPath**：包含C/C++库的路径集合，多个路径用文件分隔符分隔，可以为null。
    * **parent**：父加载器。

* 3.**PathClassLoader**

  * Android系统用来加载系统类和应用程序的类。

  * Java代码实现

    ```java
    public class PathClassLoader extends BaseDexClassLoader {
        public PathClassLoader(String dexPath, ClassLoader parent) {
            super((String)null, (File)null, (String)null, (ClassLoader)null);
            throw new RuntimeException("Stub!");
        }
        public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
            super((String)null, (File)null, (String)null, (ClassLoader)null);
            throw new RuntimeException("Stub!");
        }
    }
    ```

    没有参数`optimizedDirectory`，因为默认了参数`optimizedDirectory`的值为`/data/dalvik-cache`。很显然PathClassLoader无法定义解压的dex文件存储路径，因此PathClassLoader通常用来加载已经安装的apk的dex文件（安装的apk的dex文件会存储在`/data/dalvik-cache`中）。

### 2.2 ClassLoader的继承关系

运行一个Android应用程序需要用到几种类型的类加载器？

```java
public class MainActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ClassLoader loader = MainActivity.class.getClassLoader();
        while (loader != null) {
            Log.i("niejianjian", loader.toString()); // 1
            loader = loader.getParent();
        }
    }
}
```

打印结果为：

```
01-10 11:14:01.333 27044-27044/com.nfkj.haoxian I/niej: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.nfkj.haoxian-1.apk"],nativeLibraryDirectories=[/data/app-lib/com.nfkj.haoxian-1, /vendor/lib, /data/cust/lib, /system/lib]]]
01-10 11:14:01.333 27044-27044/com.nfkj.haoxian I/niej: java.lang.BootClassLoader@41676818
```

* 用到两种类加载器，一种是`PathClassLoader`，另一种是`BootClassLoader`。
* `/data/app/com.nfkj.haoxian-1.apk`是应用安装在手机上的位置。
* `DexPathList`是在`BaseDexClassLoader`的构造方法中创建的，里面存储了dex相关文件路径。

除了3中主要的类加载器，Android还提供了其他的类加载器，继承关系图如下：

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/classloaderextendsrelation_android8.png" alt="Androi 8.0 中的ClassLoader的继承关系 " style="zoom:70%;" />

图中一共有8个相关类，介绍如下：

* **ClassLoader**是一个抽象类，其中定义了ClassLoader的主要功能。**BootClassLoader**是它的内部类。
* **SecureClassLoader**类和JDK8中的SecureClassLoader类的代码是一样的，它继承了抽象类ClassLoader。SecureClassLoader并不是ClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。
* **URLClassLoader**类和JDK8中的URLClassLoader类的代码是一样的，它继承自SecureClassLoader，用来通过URL路径从jar文件和文件夹中加载类和资源。
* **InMemoryDexClassLoader**是Android 8.0新增的类加载器，继承自BaseDexClassLoader，用于加载内存中的dex文件。
* **BaseDexClassLoader**继承自ClassLoader，是抽象类ClassLoader的具体实现类，PathClassLoader、DexClassLoader和InMemoryDexClassLoader都继承自它。

### 2.3 ClassLoader的加载过程

Android的ClassLoader同样遵循双亲委托模式，ClassLoader的加载方法为`loadClass`方法。定义在抽象类ClassLoader中，如下所示：

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    Class<?> c = findLoadedClass(name); // 1
    if (c == null) {
        try {
            if (parent != null) { // 2
                c = parent.loadClass(name, false);
            } else {
                c = findBootstrapClassOrNull(name); // 3
            }
        } catch (ClassNotFoundException e) { }
        if (c == null) { // 4
            c = findClass(name); // 5
        }
    }
    return c;
}
```

上述代码展示了双亲委托的过程。ClassLoader的`findClass`方法如下：

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```

`findClass`直接抛出了异常，说明`findClass`方法需要子类来实现。`BaseDexClassLoader`的代码如下：

```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
                          String librarySearchPath, ClassLoader parent) {
    super(parent);
    this.pathList = new DexPathList(this, dexPath, librarySearchPath, null); 
    ...
}
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    Class c = pathList.findClass(name, suppressedExceptions); // 1
    ...
}
```

在`BaseDexClassLoader`的构造方法中创建了`DexPathList`，注释1处调用了`DexPathList`的`findClass`方法：

```java
public Class<?> findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) { // 1
        Class<?> clazz = element.findClass(name, definingContext, suppressed); // 2
        if (clazz != null) { return clazz; }
    }
    ...
}
```

注释1处遍历`Element`数组，注释2处调用`Element`的`findClass`方法。`Element`是`DexPathList`的静态内部类：

```java
/*package*/ static class Element {
    private final File path;
    private final DexFile dexFile;
    private ClassPathURLStreamHandler urlHandler;
    private boolean initialized;
    public Element(DexFile dexFile, File dexZipPath) {
        this.dexFile = dexFile;
        this.path = dexZipPath;
    }
    public Element(DexFile dexFile) {
        this.dexFile = dexFile;
        this.path = null;
    }
    public Element(File path) {
        this.path = path;
        this.dexFile = null;
    }
    
    public Class<?> findClass(String name, ClassLoader definingContext,
                              List<Throwable> suppressed) {
        return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed)
                : null; // 1
    }
}
```

从`Element`的构造方法可以看出，其内部封装了`DexFile`，它用于加载dex。注释1处如果`DexFile`不为null，就调用`DexFile`的`loadClassBinaryName`方法，随后方法内部又调用了`defineClass`方法，之后调用了`defineClassNative`方法，这个方法是一个native方法。

### 2.4 BootClassLoader的创建

`ZygoteInit`的`main`方法中，有如下一行代码：

```java
preload(bootTimingsTraceLog);
```

在`preload`方法内部又调用了`prelaodClasses`方法，简略如下：

```java
private static final String PRELOADED_CLASSES = "/system/etc/preloaded-classes";
private static void preloadClasses() {
    InputStream is;
    try {
        is = new FileInputStream(PRELOADED_CLASSES); // 1
    } catch (FileNotFoundException e) {
        return;
    }
    ...
    try {
        BufferedReader br
                = new BufferedReader(new InputStreamReader(is), 256); // 2
        int count = 0;
        String line;
        while ((line = br.readLine()) != null) { // 3
            ...
            try {
                Class.forName(line, true, null); // 4
                count++;
            } 
        }
    }
}
```

该方法用于Zygote进程初始化时预加载常用类。

* 注释1处将`/system/etc/preloaded-classes`文件封装成FileInputStream，该文件中存有预加载类的目录。
* 注释2处将FileInputStream封装为BufferedReader。
* 注释3处进行遍历BufferReader，读出所有预加载类的名称。
* 注释4处用`forName`方法对所有读出的预加载类进行加载。

```java
public static Class<?> forName(String name, boolean initialize,
                               ClassLoader loader)
        throws ClassNotFoundException {
    if (loader == null) {
        loader = BootClassLoader.getInstance(); // 1
    }
    Class<?> result;
    try {
        result = classForName(name, initialize, loader); // 2
    } catch (ClassNotFoundException e) {
        Throwable cause = e.getCause();
        if (cause instanceof LinkageError) {
            throw (LinkageError) cause;
        }
        throw e;
    }
    return result;
}
```

注释1处创建了`BootClassLoader`，并将该实例传入到了注释2处的`classForName`方法中，该方法是Native方法。

### 2.5 PathClassLoader的创建

* Zygote进程启动SystemServer进程时会调用`ZygoteInit`的`startSystemServer`方法

* `startSystemServer`方法中，当`pid==0`也就是代码在新创建的SystemServer中运行的时候，执行`handleSystemServerProcess`方法。

* 然后调用`createPathClassLoader`方法，

* 调用`PathClassLoaderFactory.createClassLoader`，在该方法中有如下代码：

  ```java
  PathClassLoader pathClassLoader = new PathClassLoader(dexPath, ...);
  ```

  

