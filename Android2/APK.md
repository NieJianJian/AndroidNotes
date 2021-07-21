## APK的构建过程

### 1. APK的构建过程主要分为以下几个步骤：

1. 通过 AAPT（Android Asset Packaging Tool）打包 res 资源文件，比如 AndroidManifest.xml 和 xml 布局文件，将这些xm布局文件编译为二进制，其中 assets 文件夹和 raw 文件不会被编译为二进制，最终会生成 R.java 文件和 resources.arsc 文件。
2. AIDL 工具会将所有aidl接口转化为对应的 Java 接口。
3. 所有的 Java 代码，包括 R.java 文件和 Java接口都会被 Java 编译器编译成 .class 文件。
4. Dex 工具会将第（3）步生成的 .class 文件、第三方库和其他 .class 文件都编译成 .dex 文件。
5. 第（4）步编译生成的 .dex 文件、编译过的资源、无需编译的资源都（如图片等）会被 ApkBuilder 工具打包成 apk 文件。
6. 使用 Debug Keystore 或者 Release Keystore 对第（5）步生成的 apk 文件进行签名。
7. 如果是对 Apk正式签名，那么还需要使用 zipalign 工具对 APK 进行对齐操作，这样应用运行时会减少内存的开销。

![APK的构建过程](https://upload-images.jianshu.io/upload_images/2846231-65d53c4741f6354a.png?imageMogr2/auto-orient/strip|imageView2/2/w/878/format/webp)

概括步骤如下：

编译 > dex -> 打包成apk -> 签名和对齐

### 2. APK解压后的结构

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

- **AndroidManifest.xml**：二进制格式，并且合并了所有子项目的AndriodManifest.xml文件。
- **META-INF**：存放签名相关文件，校验APK中所有文件的合法性，识别唯一开发者，不让其他人随便反编译修改你的APK文件。
- **classes.dex**：Android项目中所有Java代码最终编译后的形式。不管是开发者自己写的Java代码还是依赖的第三方库，都会由编译器经过* .java -> * .class -> * .dex的转换变为Android虚拟机可以识别运行的dex格式文件。这和传统的JVM应用（桌面级的Java应用、服务端的Java应用，这类基于Oracle官方Java虚拟机的应用）是不同，传统的JVM直接运行.class格式的文件，而Android是Google自己定制的Dalvik/Art虚拟，所以运行的是对.class格式进行压缩合成的.dex格式文件。Kotlin或者其他JVM上的语言，都是先转为class文件，再合并成dex文件。
- **resources.arsc**：包含所有资源的ID，以及具体的ID值和类型信息。平时使用的R.XX这类ID，实际上都是一个由32为数字组成的资源ID。它们有各种不同类型，要么字符串，要么数字，要么是资源路径。而如果是资源路径，则必然指代的是res文件中的资源文件。arsc文件只是ID信息，而实际资源内容，像是XML文件或者图片文件，都是放在res目录下，并且编译期间做过一些压缩处理。*在程序运行过程中，会先从arsc文件中找到相应的ID所对应的资源路径，然后再访问实际的res文件中路径所在的资源。*
- **assets**：存放资源，不带资源ID的原始文件，未压缩。直接指定路径访问资源。
- **lib**：存放JNI相关的so库，所有C/C++/Asm代码编译后的最终二进制形式文件。

### 3. 拓展

* aapt相关使用

  * noCompress

    描述：是否对资源进行压缩，默认不对"jpg"、"png"压缩。如果传入’’，则表明全部资源不会进行压缩。

    ```java
    aaptOptions{
        // 不对 bat 进行压缩
    	noCompress '.bat'
    }
    ```

    压缩后的资源可以通过 `aapt l -v apk路径` 进行查看压缩的细节。

    [aaptOptions——安卓gradle](https://blog.csdn.net/weixin_37625173/article/details/103685230)

