## 64K问题(MultiDex 分包)

* **Android为什么分Dex （MultiDex）**

  ​        因为在ART以前的Android系统中,Dex文件对于方法索引是用一个short类型的数据来存放的.而short的最大值是65535,因此当项目足够大包含方法数目足够多超过了65535(包括引用的外部Lib里面的所有方法),当运行App,就会得到如下的错误提示：

  ```
  Unable to execute dex: method ID not in [0, 0xffff]: 65536
  Conversion to Dalvik format failed: Unable to execute dex: method ID not in [0, 0xffff]: 65536
  ```

* **方法一：突破65535限制**

  方法数突破65535之后，需要dex分包，配置如下：

  ```java
  defaultConfig {
      /**添加多 dex分包支持*/
      multiDexEnabled true
  }
  ```

  ```java
  dependencies {
      compile 'com.android.support:multidex:1.0.2' 
  }
  ```

* **方法二：minSdkVersion 设置 21及以上**

  ​        Android 5.0 之后，安卓系统采用的是ART虚拟机，如果方法超过65535个，会自动分包，天然支持有多个dex文件，ART 在应用安装时执行预编译，将多个dex文件合并成一个oat文件执行。所以minSdkVersion参数设置为21及其以上时，就无需配置MultiDex库了。

* **出现的问题**

  ```java
  Caused by: java.lang.ClassNotFoundException: Didn't find class "android.support.v4.content.FileProvider" on path: DexPathList[[zip file "/data/app/lanyue.reader.big-2.apk"],nativeLibraryDirectories=[/data/app-lib/lanyue.reader.big-2, /vendor/lib, /data/cust/lib, /system/lib]]
  ```

  ***环境：***运行在Android 4.4.4 (API 19)的设备上出现以上错误。Android5.0 (API 21)以上的设备，运行正常。

  ***解决方法***：

  1.Applicatiaon 继承自 MultiDexApplication

  2.重写attachBaseContext方法

  ```
      @Override
      protected void attachBaseContext(Context base) {
          super.attachBaseContext(base);
          MultiDex.install(this);
      }
  ```

  ***缺点***：会导致dex过程变慢

* **原因**

  Android 5.0 (API 21)及以上版本使用 art 支持多 dex，而低版本dalvik默认先加载主 dex ，如果启动时需要的类不在主dex中内就会报ClassNotFoundException

* **MultiDex Support Lib方案的局限性**

  * 在应用安装到手机上的时候dex文件的安装是复杂的(complex)有可能会因为第二个dex文件太大导致ANR。请用proguard优化你的代码。
  * 使用了mulitDex的App有可能在4.0(api level 14)以前的机器上无法启动，因为Dalvik linearAlloc bug(Issue [22586](https://code.google.com/p/android/issues/detail?id=22586)) 。请多多测试自祈多福。用proguard优化你的代码将减少该bug几率。
  * 使用了mulitDex的App在runtime期间有可能因为Dalvik linearAlloc limit (Issue [78035](https://code.google.com/p/android/issues/detail?id=78035)) Crash。该内存分配限制在 4.0版本被增大，但是5.0以下的机器上的Apps依然会存在这个限制。
  * 主dex被dalvik虚拟机执行时候，哪些类必须在主dex文件里面这个问题比较复杂。build tools 可以搞定这个问题。但是如果你代码存在反射和native的调用也不保证100%正确。

* **参考链接**

  [Android 分Dex(MultiDex)](<https://www.cnblogs.com/wingyip/p/4496028.html>)

  [multiDexEnabled遇坑简记](<https://www.jianshu.com/p/cddfc89ce947>)

  [MultiDex到底有多坑](<https://www.cnblogs.com/lzl-sml/p/5216861.html>)

  [Android MultiDex](<https://www.cnblogs.com/CharlesGrant/p/5112597.html>)

  [其实你不知道MultiDex到底有多坑](<http://blog.zongwu233.com/the-touble-of-multidex/>)