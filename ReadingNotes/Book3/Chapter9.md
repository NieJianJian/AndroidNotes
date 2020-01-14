## 第9章 JNI原理

一般需要用到JNI技术的情况：

* 需要调用Java语言不支持的依赖于操作系统平台特性的一些功能。例如：需要调用当前UNIX系统的某个功能，而Java不支持这个功能。
* 为了整合一些以前的非Java语言开发的系统。

* 为了节省程序的运行时间，必须采用其他语言来提升运行效率。例如：游戏、音视频开发涉及的音视频解码和图像绘制需要更快的处理速度。

***

### 1 MediaRecorder框架中的JNI

MediaRecorder用于录音和录像。MediaRecorder框架中的JNI，如下图所示：

![MediaRecorder框架中的JNI](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/jni_mediarecorderframework.png)

* Java Framework层对应的是`MediaRecorder.java`，也就是我们在应用开发中直接调用的类。
* JNI层对应的是`libmedia_jni.so`，它是一个JNI的动态库。
* Native层对应的是`libmedia.so`，这个动态库完成了实际的调用功能。

### 1.1 Java Framework层的MediaRecorder

我们先来看`MediaRecorder.java`的源码，截取部分和JNI有关的源码如下所示：

```java
public class MediaRecorder {
    static {
        System.loadLibrary("media_jni"); // 1
        native_init(); // 2
    }
    private static native final void native_init(); // 3
    public native void start() throws IllegalStateException;
}
```

* 注释1处用来加载名为`media_jni`的动态库，也就是`libmedia_jni.so`。
* 注释2处的`native_init`方法内部会调用Native方法，用来完成JNI的注册。

### 1.2 JNI层的MediaRecorder

MediaRecorder的JNI层是由`android_media_MediaRecorder.cpp`实现。

### 1.3 Native方法注册

Native方法注册分为**静态注册**和**动态注册**。静态注册多用于NDK开发，动态注册多用于Framework开发。

* 1.**静态注册**

  在AS中新建一个Java Library，命名为media，仿照系统写一个简单的`MediaRecorder.java`

  ```java
  public class MediaRecorder {
      static {
          System.loadLibrary("media_jni");
          native_init();
      }
      private static native final void native_init();
      public native void start() throws IllegalStateException;
  }
  ```

  进入到项目的`media/src/main/java`目录中执行如下命令：

  ```
  javac com/example/MediaRecord.java
  javah com.example.MediaRecord
  ```

  第二个命令会在当前目录（media/src/main/java）中生成`com_example_MediaRecord.h`文件，该文件中包括如下部分代码：

  ```c++
  JNIEXPORT void JNICALL JAVA_com_example_MediaRecorder_native_1init
    (JNIEnv *, jclass); 
  ```

  Java中的`native_init`方法被声明为上述代码，也就是`JAVA_com_example_MediaRecorder_native_1init`，

  * 上述方法名中多了一个"1"，是因为Java的`native_init`方法中包含了"_ "，转换成JNI方法后会编程" _1"
  * `JNIEnv`是Native世界中Java环境代表，通过`JNIEnv * `指针就可以在Native世界中访问Java世界的代码进行操作。**只在创建它的线程中有效，不能跨进程传递**。
  * `jclass`是JNI的数据类型，对应Java的`java.lang.Class`实例。

  Java中调用`native_init`方法时，会找到`JAVA_com_example_MediaRecorder_native_1init`并建立关联，其实是保存JNI的函数指针，这样再次调用`native_init`犯法时直接使用这个函数指针就可以了。

  静态注册是根据方法名，将Java方法和JNI函数建立关联。缺点如下：

  * JNI层的函数名称过长。
  * 声明Native方法的类需要用javah生成头文件。
  * 初次使用Native方法需要建立关联，影响效率。

* 2.**动态注册**

  静态注册是Java的Native方法通过方法指针来与JNI进行关联，如果Java的Native方法知道它的JNI中对应的函数指针，就可以避免静态注册的缺点，这就是动态注册。

  * `JNINativeMethod`用来记录Java的Native方法和JNI方法的关联关系，在`jni.h`中被定义：

    ```c++
    typedef struct {
        const char* name; // Java方法名字
        const char* signature; // Java方法的签名
        void*       fnPtr; // JNI中对应方法的指针
    } JNINativeMethod;
    ```

  * 系统`MediaRecorder`采用的就是动态注册

    ```c++
    // @frameworks/base/media/jni/android_media_MediaRecorder.cpp
    static const JNINativeMethod gMethods[] = {
        ......
        {"start",         "()V", (void *)android_media_MediaRecorder_start},
        {"stop",          "()V", (void *)android_media_MediaRecorder_stop},
        {"native_init",   "()V", (void *)android_media_MediaRecorder_native_init},
        ......
    };
    ```

    * `gMethods`数组存储的是MediaRecorder的Native方法与JNI层函数的对应关系。
    * `start`是Java层的Native方法，
    * `()V`是start方法的签名信息，
    * `android_media_MediaRecorder_start`是JNI层的函数。

  * 只定义`JNINativeMethod`类型的数组是不行的，还需要注册它，

    * 注册函数为`register_android_media_Mediarecorder`，

    * 该注册函数调用在`android_media_MediaPlayer.cpp`的`JNI_OnLoad`函数中。

    * `JNI_OnLoad`函数调用在`System.loadLibrary`函数后。因为多媒体框架中的很多框架都进行`JNINativeMathod`数组注册，因此，注册函数就被统一定义在`android_media_MediaPlayer.cpp`的`JNI_OnLoad`函数中，如下所示：

      ```c++
      jint JNI_OnLoad(JavaVM* vm, void* /* reserved */)
      {
          JNIEnv* env = NULL;
          ......
          if (register_android_media_MediaPlayer(env) < 0) { // 1
              ALOGE("ERROR: MediaPlayer native registration failed\n");
              goto bail;
          }
          if (register_android_media_MediaRecorder(env) < 0) { // 2
              ALOGE("ERROR: MediaRecorder native registration failed\n");
              goto bail;
          }
          ......
      }
      ```

      随后调用`AndroidRuntime`的`registerNativeMethods`函数，

      随后又调用了定义在`JNIHelp.cpp`中的`jniRegisterNativeMethods`函数。该方法中：

      ```c++
      if((*env)->RegisterNatives(e, c.get(),gMethods,numMethods) < 0){
      ```

      如上述代码，最终调用到了`JNIEnv`的`RegisterNatives`函数类完成JNI的注册。

***

### 2 数据类型的转换

### 2.1 基本数据类型的转换

基本数据类型的转换表如下所示：

| Java    | Native   | Signature |
| ------- | -------- | --------- |
| byte    | jbyte    | B         |
| char    | jchar    | C         |
| double  | jdouble  | D         |
| float   | jfloat   | F         |
| int     | jint     | I         |
| short   | jshort   | S         |
| long    | jlong    | J         |
| boolean | jboolean | Z         |
| void    | void     | V         |

除了void，其他的数据类型只需要在前面加上"j"就可以了。

### 2.2 引用数据类型的转换

数组的JNI层数据类型需以"Array"结尾，签名格式的开头都会有"["，有些数据类型的签名以";"结尾，引用数据类型还有继承关系。引用数据类型的转换表如下所示：

| Java      | Native        | Signture              |
| --------- | ------------- | --------------------- |
| 所有对象  | jobject       | L+classname+;         |
| Class     | jclass        | Ljava/lang/class;     |
| String    | jstring       | Ljava/lang/String;    |
| Throwable | jthrowable    | Ljava/lang/Throwable' |
| Object[]  | jobjectArray  | [L+classname+;        |
| byte[]    | jbyteArray    | [B                    |
| char[]    | jcharArray    | [C                    |
| double[]  | jdoubleArray  | [D                    |
| float[]   | jfloatArray   | [F                    |
| int[]     | jintArray     | [I                    |
| short[]   | jshortArray   | [S                    |
| long[]    | jlongArray    | [J                    |
| boolean[] | jbooleanArray | [Z                    |

jclass、jstring、jarray和jthrowable都继承jobject，而jobjectArray、jintArray和jlongArray等类型继承jarray。

***

### 3 方法签名

方法签名就是由签名格式组成的，之前的`gMethods`数组中，"()V"这种就是方法签名。

**参数类型和返回值类型组合在一起作为方法签名**。

JNI的方法签名格式为：

> （参数签名格式...）返回值签名格式

以`gMethods`数组的`native_setup`函数举例，Java中是如下定义的：

```java
private native final void native_setup(Object mediarecorder_this,
            String clientName, String opPackageName) throws IllegalStateException;
```

它在JNI中的方法签名为：

```
(Ljava/lang/Object;Ljava/lang/String;Ljava/lang/String;)V
```

`javap`命令可以自动生成方法签名：

```
javap -s -p ***.class
```

`-s`表示输出内部类型签名，`-p`表示打印处所有的方法和成员（默认打印public成员）。

***

### 4 解析JNIEnv

JNIEnv是Native世界中Java环境的代表，通过JNIEnv *指针就可以在Native世界中访问Java世界的代码进行操作，它只在创建它的线程中有效，不同线程的JNIEnv是彼此独立的。JNIEnv的主要作用是以下两点：

* 调用Java的方法
* 操作Java（操作Java中的变量和对象等）

***

### 5 引用类型

JNI的引用类型分为本地引用、全局引用和弱全局引用。

### 5.1 本地引用

JNIEnv提供的函数所返回的引用基本都是本地引用，也会JNI中最常见的引用类型，特点如下：

* 当Native函数返回时，这个本地引用会自动释放。也可以使用JNIEnv的`DeleteLocalRef`函数手动删除。
* 只在创建它的线程中有效，不能够跨线程使用。
* 局部引用是JVM负责的引用类型，受JVM管理。

### 5. 全局引用

全局引用和本地引用几乎是相反的，特点如下：

* 在Native函数返回时不会被自动释放，因此全局引用需要手动来进行释放，并且不会被GC回收。
* 全局引用是可以跨线程使用的。
* 全局引用不受到JVM管理
* JNIEnv的`NewGlobalRef`函数用来创建全局引用，`DeleteGlobalRef`函数用来释放。

### 5.3 弱全局引用

弱全局引用是一种特殊的全局引用。

* 可以被GC回收，弱全局引用被GC回收之后会指向NULL。

* JNIEnv的`NewWeakGlobalRef`函数用来创建弱全局引用，`DeleteWeakGlobalRef`函数用来释放。

* 使用前需要判断是否被回收，使用JNIEnv的`IsSameObject`函数判断。

  ```c++
  if (env->IsSameObject(weakGlobalRef, NULL))
  ```

  