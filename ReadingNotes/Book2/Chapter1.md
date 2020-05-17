## 第1章 热修复技术介绍

### 1 基本概念

　　Android安装包是一个APK文件，实际上是zip格式封装，解压后内容大致如下：

```
AndroidManifest.xml
META-INF/CERT.RSA
META-INF/CERT.SF
META-INF/MANIFEST.MF
classes.dex
resources.arsc
res/drawable/a.png
res/drawable/b.xml
res/layout/activity_main.xml
assets/c.png
lib/armeabi/libnative-lib.so
lib/x86/libnative-lib.so
```

* APK文件格式介绍
  * **AndroidManifest.xml**：二进制格式，并且合并了所有子项目的AndriodManifest.xml文件。
  * **META-INF**：存放签名相关文件，校验APK中所有文件的合法性，识别唯一开发者，不让其他人随便反编译修改你的APK文件。
  * **classes.dex**：Android项目中所有Java代码最终编译后的形式。不管是开发者自己写的Java代码还是依赖的第三方库，都会由编译器经过* .java -> * .class -> * .dex的转换变为Android虚拟机可以识别运行的dex格式文件。这和传统的JVM应用（桌面级的Java应用、服务端的Java应用，这类基于Oracle官方Java虚拟机的应用）是不同，传统的JVM直接运行.class格式的文件，而Android是Google自己定制的Dalvik/Art虚拟，所以运行的是对.class格式进行压缩合成的.dex格式文件。Kotlin或者其他JVM上的语言，都是先转为class文件，再合并成dex文件。
  * **resources.arsc**：包含所有资源的ID，以及具体的ID值和类型信息。平时使用的R.XX这类ID，实际上都是一个由32为数字组成的资源ID。它们有各种不同类型，要么字符串，要么数字，要么是资源路径。而如果是资源路径，则必然指代的是res文件中的资源文件。arsc文件只是ID信息，而实际资源内容，像是XML文件或者图片文件，都是放在res目录下，并且编译期间做过一些压缩处理。*在程序运行过程中，会先从arsc文件中找到相应的ID所对应的资源路径，然后再访问实际的res文件中路径所在的资源。*
  * **assets**：存放资源，不带资源ID的原始文件，未压缩。直接指定路径访问资源。
  * **lib**：存放JNI相关的so库，所有C/C++/Asm代码编译后的最终二进制形式文件。
* App本质上如何实现热修复功能
  * **AndroidManifest修复**：出现BUG无法修复，因为它是由系统进行解析的。系统会直接获取安装包中唯一的AndroidManifest文件，解析过程中不会访问补丁包文件。**因此四大组件是不能直接添加的**。
    * **Q：如何在AndroidManifest里面新增四大组件？**
    * **A**：预先在安装包的AndroidManifest里面埋入代理的组件，在每次新增组件时，进行偷梁换柱，通过预埋的代理组件实现与系统进程间的通信。
  * **代码修复**：需要在补丁包里包含一个新逻辑的dex文件；然后在程序运行时加载这个dex文件，并且改变执行流，从执行原有安装包立的classes.dex文件引导到执行新dex文件。
  * **资源修复**：正常情况下，资源包就是整个APK安装包，如果想要新增一个原有安装包里不存在的资源，就得把原有的安装包替换为新资源包，或把新的资源包插入程序的查找过程。*如桌面图标、通知栏图标、RemoteView之类的资源，是由系统直接解析安装包里的资源得到的，是任何热修复方案都无法进行资源替换和修复。*
  * **so库修复**：so库由System.load加载， 只需要在加载的时候优先加载补丁包的so库就可以了。

### 2 技术积淀

优秀的热修复方案：

* 阿里巴巴手淘团队Sophix
* 腾讯QQ空间超级补丁技术
* 微信的Tinker
* 饿了么Amigo
* 美团Robust

### 3 技术概览

　　Sophix的核心设计理念，就是非侵入性。

　　Sophix方案中，唯一需要做的就是**初始化和请求补丁的两行代码**。

### 3.1 代码修复

* 代码修复两大主要方案：
  * **阿里系的底层替换方法**：限制多，但时效性最好，加载轻快，立即见效。
  * **腾讯系的类加载方法**：时效性差，需要冷启动，但修复范围广，限制少。

* **（1）底层替换方案**

  　　在已经加载的类中直接替换原有方法，无法实现对原有类进行方法和字段的增减，这样会破坏类结构。类中方法个数的增减，会导致这个类以及整个dex的方法个数变化，导致无法正常索引。

    　　传统底层替换方式，是依赖直接修改虚拟机方法实体的具体字段实现的。如Dalvik方法的JNI函数指针、修改类或方法的访问权限等。　　

* **（2）类加载方案**

  　　**原理**：在App重新启动后让Classloader去加载新的类。*Android无法对一类进行卸载*，所以需要重启。

    　　dex比较的最佳粒度，应该是类的维度。全量合成dex的技术，直接利用Android原有的类查找与合成机制，快速合成新的全量dex文件。

    　　我们重新编排了包中dex文件的顺序。在虚拟机查找类的时候，会优先查找classes.dex中的类，然后才是classes2.dex、classes3.dex，也可以看作是dex文件级别的类插桩方案。这个方式它对旧包与补丁包中的classes.dex顺序进行了打破与重组，最终使系统可以自然地识别到这个顺序，以实现类覆盖的目的。

* **（3）Sophix**

  　　Sophix涵盖了底层替换和类加载两种方案的优点。

  * 针对小修改，在底层替换方案限制范围内，使用底层替换
  * 代码修改超过底层替换限制，使用类加载替换方案
  * 运行时阶段，Sophix还会再次判断所运行机型是否支持热修复。

### 3.2 资源修复

Instant Run中资源热修复分为两步：

* 1.构造一个新的AssetManager，并通过反射调用addAssetPath函数，然后把完整的新资源包加载到AssetManager中。这样就得到了一个含有所有新资源的AssetManager。
* 2.找到所有之前引用到原有AssetManager的地方，通过反射，把引用处替换为新的AssetManager。

　　Instant Run大量代码都是在*处理兼容性问题*和*搜索到AssetManager所有的引用处*。

Sophix方案：

* **原理**：构造一个package id为0x66（不与目前已加载的地址为0x7f的包冲突）的资源包，这个包里只包含需要改变的资源项，然后直接在原有AssetManager中通过addAssetPath函数添加这个包就可以了。

* **优势：**
  * 不修改AssetManager的引用处，替换资源更快更安全。
  * 不必下发完整包，补丁包中包含有变动得资源
  * 不需要在运行时合成完整包，不占用运行时计算和内存资源。

### 3.3 so库修复

　　so库的修复本质上是对native方法的修复和替换。

　　Sophix采用的是类似类修复反射注入方式。把补丁so库的路径插入到nativeLibraryDirectories数组的最前面，就能够达到加载so库的时候是补丁so库的目录从而修复BUG的目的。

　　