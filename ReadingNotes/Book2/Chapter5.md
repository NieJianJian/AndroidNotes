## 第5章 so库热修复技术

### 1 so库加载原理

### 1.1 加载方式

Java API 提供以下两个接口加载一个so库：

* **`System.loadLibrary(String libName)`**：传进去的参数是so库名称，表示so库文件，位于APK压缩文件的libs目录，最后复制到APK安装目录下。
* **`System.load(String pathName)`**：传进去的参数是so库在磁盘中的完整路径，加载一个自定义外部so库文件。

上面两种方法，最终都调用`nativeLoad`这个native方法去加载so库，这个方法的参数`fileName:so`是库在磁盘中的完整路径名。

### 1.2 注册方式

native方法有动态注册和静态注册两种方式。

```java
public class MainActivity extends Activity {
    static {
        System.loadLibrary("jnitest");
    }
    public static native String stringFromJNI();
    public static native void test;
}

//静态注册stringFromJNI方法
extern "C" jstring Java_com_test_example_MainActivity_stringFromJNI(
        JNIEnv *env, jclass clazz) {
    std::string hello = "jni stringFrom old....";
    return env->NewStrignUTF(hello.c_str());
}

//动态注册test方法
void test(JNIEnv *env, jclass clazz) {
    LOGD(jni test old .....);
}
JNINativeMethod nativeMethods[] = {
    {"test" , "()V", (void*) test}
};
#define JNIREG_CLASS "com/test/example/MainActivity";
JNIEXPORT jnint JNICALL JNI_OnLoad(javaVM *vm, void *reserved) {
    LOGD("lod JNI_OnLoad");
    ...
    jclass claz = env->FindClass(JNIREG_CLASS);
    if(env->RegisterNatives(claz, nativeMethods,
        sizeof(nativeMethods)/sizeof(nativeMethods[0])) != JNI_OK)) {
        return JNI_ERR;
    }
    return JNI_VERSION_1_4;
}
```

* 静态注册的native方法必须是**`"Java_"`+类完整路径+方法名**的格式。
* 动态注册的antive方法必须实现`JNI_OnLoad`方法，同时实现一个`JNINativeMethod[]`数组。

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/so_native_method_yingshe.png" alt="native方法映射逻辑 " style="zoom:50%;" />

总结如下：

* 静态注册的native方法映射是在该`native`方法第一次执行的时候才完成映射，当然前提是该so库已经加载。

* 动态注册的native方法映射通过加载so库过程中调用`JNI_OnLoad`方法调用完成。

***

### 2 so库热部署实时生效的可行性分析

### 2.1 动态注册native方法实时生效

　　动态注册的native方法调用一次JNI_OnLoad方法都会重新完成一次映射，所以只要先加载原来的so库，再加载补丁so库，就能完成Java层native方法到native层patch后的新方法映射，这样就能完成动态注册native方法的patch实时修复。如图所示：

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/so_dynamicregister_native.png)

　　**问题**：实测发现art下实时生效，但是Dalvik下做不到实时生效。实际上Dvlvik下第二次load补丁so库，执行的仍然是原来so库的`JNI_OnLoad`方法，首先怀疑以下两个函数是否有问题。

* **`dlopen()`**：返回给我们一个动态链接库的句柄。
* **`dlsym()`**：通过一个dlopen得到的动态链接库句柄，来超找一个symbol。

　　**原因**：通过代码分析，发现**Dalvik**虚拟机下是通过`bname`去做查找，传进来的参数`name`实际上是so库所在磁盘的完整路径，比如此时修复so库的路径为`/data/data/com.taobao.jni/files/libnative-lib.so`。但是此时是通过`bname:libnative-lib.so`作为key去查找的，我们知道第一次加载原来的so库`System.loadLibrary("native_lib")`实际上已经在`solist`表中存在了`native-lib`这个key，所以Dalvik下加载修复后的补丁`so`拿到的还是原来的`so`库文件的句柄。**Art**下不存在这个问题，是因为Art下这个地方以`name`作为key去查找而不是`bname`，所以art下重新加载一边补丁`so`库，拿到的是补丁`so`库的句柄。

　　**解决方案**：Dalvik下对补丁中的so进行改名，比如改为`libnative_lib-123333.so`，后面一串数字是时间戳，确保bname是全局唯一的。

### 2.2 静态注册native方法的实时生效

　　**问题**：静态注册native方法的映射在native方法第一次执行的时候就完成了，所以如果native方法在加载补丁so库之前已经执行过了，怎么办？

　　**解决方案**：JNI API提供了解注册的接口。`UnregisterNatives`函数把`jclazz`所在类的所有native方法都重新指向`dvmResolveNativeMathod`，所以调用`UnregisterNatives`之后不管是静态注册还是动态注册的native方法之前是否执行过，在加载补丁so的时候都会重新去做映射。

> hashtable的实现源码在`dalvik/vm/Hash.h`和`dalvik/vm/Hash.cpp`文件中，hashtable的遍历和插入都是在`dvmHashTableLookup`方法中实现，`java.hashtable`和`c.hashtable`的异同点如下：
>
> * 公共点：两者都是**数组实现**，hashtable容量如果超过默认值都会进行扩容，都是对`key`进行`hash`计算。然后跟`hashtable`的长度进行取模作为`bucket`。
> * 不同点：Dalvik虚拟机下hashtable put/get操作实现方法，实际上实现要比java hashmap的实现简单一些，java hashmap的put实现需要处理hash 冲突的情况，一般情况下会通过在**冲突节点上新增一个链表处理冲突，然后`get`实现会遍历这个链表通过`equals`方法比较`value`是否一致进行查找**，dalvik下hashtable的put实现上（doAdd=true）只是简单的把**指针下移知道下一个空节点**。get实现（doAdd=false）首先根据hash值计算处bucket位置，然后通过`cmpFunc`函数比较值是否一致，不一致，指针下移。hashtable的遍历实际上就是数组遍历实现。

**难点**：native方法的修改是在so库中，我们的补丁工具很难检测出哪个Java类需要解注册。

**问题**：假设知道需要解注册哪个native方法，然后加载补丁so库之后，再次执行该native方法，看起来可以让该native方法实时生效，但是在补丁so库重命名的前提下，Java层native方法可能映射到原so库的方法，也可能映射到补丁so库的修复后的方法。

**原理**：源码中有一个名为`gDvm.nativeLibs`的全局变量，它是一个hashtable，存放着整个虚拟机加载so库的SharedLib结构指针。然后该变量作为参数传递给`dvmHashForeach`函数进行hashtable遍历。执行findMethodInLib函数看是否找到对应的native函数指针，如果第一个找到就直接return，不再进行下次查找。

![静态注册native方法实时生效](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/so_staticregister_native.png)

**结论**：对补丁so库进行重命名后，如果这个补丁so库在hashtable中的位置比原so库的位置考前，那么这个静态注册native方法就能够得到修复，位置如果靠后就得不到修复。

### 2.3 so库实时生效方案总结

so库的实时生效必须满足以下几点：

* so库为了兼容Dalvik虚拟机下动态注册native方法实时生效，必须对so文件进行改名。
* 针对so库静态注册native方法的实时生效，首先需要**解注册**静态注册的naive方法，这个也是难点，因为哦们很难知道so库中哪几个静态注册的native方法发生了变更。假设就算我们知道，也不能保证一定会被修复。
* 上面对补丁so进行了二次加载，那么肯定是多消耗依次本地内存，如果补丁so库勾搭，补丁so库够多，那么JNI层的OOM也不是没有可能。
* 另外一方面补丁so库如果新增了一个动态注册的方法而dex中没有相应方法，直接去加载这个补丁so文件会报`NoSuchMethodError`异常，具体逻辑在`dvmRegisterJNIMethod`中。我们知道如果dex新增一个native方法，那么就不能热部署只能冷启动剩下，所以此时补丁so库就不能第二次加载了。这种情况下so库的修复严重依赖于dex的修复方案。

***

### 3 so库冷部署重启生效方案

### 3.1 接口调用替换方案

SDK提供接口替换System默认加载so库接口：

```java
SOPatchManager.loadLibrary(String libName) -> 代替 System.loadLibrary(String libName)
```

`SOPatchManager.loadLibrary`接口加载so库的时候优先尝试去加载**SDK指令目录下的补丁so**，策略如下：

* 如果存在则加载补丁so库而不去加载APK安装目录下的so库。
* 如果不存在补丁so，那么掉哦那个`System.loadLibrary`去加载APK安装目录下的so库。

这个方案的优缺点：

* 优点：不需要对不同SDK版本进行兼容，因为所有的SDK版本都有System.loadLibrary这个接口。
* 缺点：调用方需要替换掉System默认加载so库接口为SDK提供的接口，如果是已经编译好的三方库的so库需要patch，那么很难做到接口的替换。

### 3.2 反射注入方案

分析`DexPathList.findLibrary`源码得出以下结论：

* Android SDK版本小于23时，可以采用类似类修复反射注入，只要把补丁so库的路径插入到`nativeLibraryDirectories`数组的最前面，就能够使得加载so库时加载的时补丁so库，而不是原来的so库的目录，从而达到修复的目的。
* Android SDK版本23及以上时，`findLibrary`发生了变化，我们只需要把补丁so库的完整路径作为参数构建一个Element对象，然后再插入到`nativeLibraryPathElements`数组的最前面就好了。

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/so_reflectioninjection.png" alt="反射注入实现 " style="zoom:50%;" />

* 优点：可以修复三方库的so库。同时接入方不需要像方案1一样强制侵入用户接口调用
* 缺点：需要不断得对SDK进行适配，如sdk23为分界线，findLibrary接口实现已经发生了变化。

***

### 4 如何正确复制补丁so库

<img src="https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/so_selectprimarycpuabis.png" alt="如何选择primaryCpuAbis " style="zoom:50%;" />

我们的补丁so库文件放到补丁包的libs目录下面，`libs`目录和`.dex`文件和`res`资源文件一起打包成一个压缩文件作为最后的补丁包，libs目录可能也包含多种abis目录。所以我们需要选择手机最合适的`primaryCpuAbi`，然后从libs目录下面选择这个`primaryCpuAbi`子目录插入到`nativeLibraryDirectories/nativeLibraryPathElements`数组中。

* sdk ≥ 21时，直接反射拿到ApplicationInfo对象的`primaryCpuAbi`即可。
* sdk < 21时，由于此时不支持64位，所以直接把`Build.CPU_ABI, Build.CPU_ABI2`作为`primaryCpuAbi`即可。

