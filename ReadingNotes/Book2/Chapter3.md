## 第3章 冷启动代码修复

### 1 冷启动类加载原理

### 1.1 冷启动实现方案

|      | QQ空间                                                       | Tinker                                                       |
| :--: | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 原理 | 为了解决Dalvik下unexpected dex problem异常而采用插桩的方式，单独放一个帮助类在独立的dex中让其他类调用，阻止了类被打上CLASS_ISPREVERIFIED标志从而规避问题的出现。最后加载补丁dex得到dexFile对象作为参数构建一个Element对象插入到dexElements数组的最前面 | 提供dex差量包，整体替换dex的方案。差量的方式给出patch.dex，然后将patch.dex与应用的classes.dex合并成一个完整的dex，完整dex加载得到的dexFile对象作为参数构建一个Element对象然后然后整体替换掉旧的dexElements数组 |
| 优点 | 没有合成整包，产物比较小，比较灵活                           | 自研dex差异算法，补丁包很小，dex merge成完整dex，Dalvik不影响类加载性能，Art下也不存在必须包含父类/引用类的情况 |
| 缺点 | Dalvik下影响类加载性能，Art下类地址写死，导致必须包含父类/引用类，最后补丁包很大 | dex合并内存消耗在vm heap上，容易OOM，最后导致dex合并失败     |

### 1.2 插桩实现的前因后果

　　**加载一个dex文件到本地内存的时候**，如果不存在odex文件，那么首先会执行dexopt，dexopt的入口在`dalvik/opt/OptMain.cpp`的main方法中，最后调用到`verifyAndOptimizeClass`执行真正的verify/optmize操作。

　　在第一次安装APK的时候，会对原dex执行dexopt，此时假如APK只存在一个dex，dvmVerifyClass(clazz)返回结果为true。然后APK中所有的类都会被打上`CLASS_ISPREVERIFIED`标志，接下来执行dvmOptimizeClass，类接着被打上`CLASS_ISOPTIMIZED`标志。

* **dvmVerifyCLass**：类校验，简单来说，类校验目的是为了防止校验类的合法性被篡改。此时会对类的每个方法进行校验，这里我们只需要知道如果类的所有方法中直接引用到的类（第一层关系，不会进行递归搜索）和当前类都在同一个dex中的话，dvmVerifyClass就返回true。
* **dvmOptimizeClass**：类优化，简单来说这个过程会把部分指令优化成虚拟机内部指令，比如方法调用指令`invoke-*`变成了`invoke-*-quick`，quick指令会从类的vtable表中直接获取，**vtable**简单来说就是类的所有方法的一张大表（包括继承自父类的方法）。

**Q**：假定A是补丁类（单独dex），类B中某个方法引用到A类，如果类B被打上了CLASS_ISPREVERIFIED标志，在引用A类的时候，由于属于不同的dex，就会报错。

**A**：为了解决这个问题，一个单独的无关帮助类被放到一个单独的dex中，原dex中所有类的构造函数都引用这个类，一般的实现方法都是侵入dex打包流程，利用.class字节码修改技术，在所有.class文件的构造函数中引用这个帮助类，***插桩由此而来***。

　　一个类的加载通常有三个阶段：`dvmResolveClass`、`dvmLinkCLass`、`dvmInitClass`。dvmInitClass阶段在类解析完并尝试初始化类的时候执行，这个方法主要完成父类的初始化、当前类的初始化、静态变量的初始化赋值等操作。

* 通常类的校验和优化都仅在APK第一次安装执行dexopt操作的时候进行。
* 如果类没有被打上CLASS_ISPREVERIFIED/CLASS_ISOPTIMIZED的标志，那么类的校验和优化都将在类的初始化阶段进行。

### 1.3 插桩导致类加载性能影响

　　插桩会导致所有类都非preverify，从而导致类校验和优化操作在类加载时触发。有一定的性能损耗。应用启动时，若同时加载大量类，可能会导致白屏。

### 1.4 避免插桩的QFix方案

### 1.5 Art下冷启动实现

　　为了解决Art下类地址写死问题，Tinker通过dex merge成一个全新完整的新dex整体替换掉旧的dexElements数组，事实上不需要这样做，Art虚拟机下面默认已经支持多dex压缩文件的加载。

　　Dalvik和Art对`DexFile.loadDex`尝试把一个dex文件解析加载到native中内存都发生了什么，实际上是调用了`DexFile.openDexFileNative`这个native方法。看一下native层对应的C/C++代码实现。

* Dalvik虚拟机下，尝试加载一个压缩文件的时候只会去把`classes.dex`加载到内存中，如果此时压缩文件中有多个dex，那么除了`classes.dex`之外的其他dex都会被直接忽略掉。
* Art虚拟机下优先加载primary dex也就是`classes.dex`，后续会加载其他的dex。所以补丁类只需要放到`classes.dex`中即可，后续出现在其他dex中的"补丁类"不会被加载。

　　**Art下冷启动解决方案**：我们只要把补丁`dex`命名为`classes.dex`。原`APK`中的`dex`依次命名为`classes(2,3,4...).dex`就可以了，然后一起打包为一个压缩文件。再通过`DexFile.loadDex`得到`DexFile`对象，最后用该`DexFile`对象整体替换旧的`dexElements`数组即可。

### 1.6 不得不说的其他点

　　`DexFile.loadDex`把一个dex文件解析并加载到native内存之前，如果dex不存在对应的odex：

* Dalvik下会执行`dexopt`。
* Art下会执行`dexoat`。

　　最后得到的都是一个优化后的`odex`。***实际上最后虚拟机执行的是odex而不是dex***。

　　**问题**：如果优化后的odex文件没有生成或者不完整，那么loadDex便不能在应用启动的时候进行，因为会阻塞loadDex线程，一般是主线程。

　　**方案**：把`loadDex`当做一个事务来看，如果中途打断，那么就删除odex，重启的时候如果发现存在odex文件，loadDex完知乎，反射注入/替换dexElements数组，实现打包。如果不存在odex文件，那么重启另一个子线程loadDex，重启之后再生效。

### 1.7 完整的方案考虑

**注入前被加载的类（比如Application类）肯定是不能被修复的**。

在没法应用热部署或者热部署失败的情况下，最后会应用代码冷启动重启生效方案，所以补丁是同一套。具体实施方案对Dalvik下和Art下分别做了处理：

* 在Dalvik下采用自行研发的全量dex方案
* 在Art下本质上虚拟机已经支持多dex的加载，只要把补丁dex作为主dex加载而已。

***

### 2 多态对冷启动类加载的影响

### 2.1 重新认识多态

