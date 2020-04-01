## 第3章 冷启动代码修复

### 1 冷启动类加载原理

### 1.1 冷启动实现方案

|      | QQ空间                                                       | Tinker                                                       |
| :--: | :----------------------------------------------------------- | :----------------------------------------------------------- |
| 原理 | 为了解决Dalvik下unexpected dex problem异常而采用插桩的方式，单独放一个帮助类在独立的dex中让其他类调用，阻止了类被打上CLASS_ISPREVERIFIED标志从而规避问题的出现。最后加载补丁dex得到dexFile对象作为参数构建一个Element对象插入到dexElements数组的最前面 | 提供dex差量包，整体替换dex的方案。差量的方式给出patch.dex，然后将patch.dex与应用的classes.dex合并成一个完整的dex，完整dex加载得到的dexFile对象作为参数构建一个Element对象然后整体替换掉旧的dexElements数组 |
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

　　**方案**：把`loadDex`当做一个事务来看，如果中途打断，那么就删除odex，重启的时候如果发现存在odex文件，loadDex完之后，反射注入/替换dexElements数组，实现打包。如果不存在odex文件，那么重启另一个子线程loadDex，重启之后再生效。

### 1.7 完整的方案考虑

**注入前被加载的类（比如Application类）肯定是不能被修复的**。

在没法应用热部署或者热部署失败的情况下，最后会应用代码冷启动重启生效方案，所以补丁是同一套。具体实施方案对Dalvik下和Art下分别做了处理：

* 在Dalvik下采用自行研发的全量dex方案
* 在Art下本质上虚拟机已经支持多dex的加载，只要把补丁dex作为主dex加载而已。

***

### 2 多态对冷启动类加载的影响

### 2.1 重新认识多态

　　**动态绑定**：实现多态的技术，是指在执行期间判断所引用对象的实际类型，根据其实际类型调用其相应的方法。多态一般指的是非静态非私有方法的多态，field和静态方法不具有多态性。

　　在虚拟机中加载每个类都会为这个类生成一张`vtable`表，`vtable`表就是当前类的所有`virtual`方法的一个数组，当前类和所有继承父类的`public/protected/default`方法（可以被继承的方法）就是`virtual`方法，`private/static`方法不属于这个范畴，因为不能被继承。

　　子类`vtable`的大小等于子类virtual方法数+父类vtable的大小。

* 整体复制父类vtable到子类的vtable；
* 遍历子类的virtual方法集合，如果方法原型一致，说明是重写父类方法，那么在相同索引位置处，子类重写方法覆盖掉vtable中父类的方法；
* 若方法原型不一致，那么就把该方法添加到vtable的末尾。

　　**从当前变量的引用类型而不是实际类型中查找，如果找不到，再去父类递归查找**。

### 2.2 冷启动方案限制

```java
public class Demo {
    public static void test_addMethod() {
        A obj = new A();
        obj.a_t2();
    }
}
class A {
    int a = 0;
    // 新增a_t1方法
    void a_t1() {
        Log.d("Sophix", "A a_t1");
    }
    void a_t2() {
        Log.d("Sophix", "A a_t2");
    }
}
```

　　修复后的APK新增了`a_t1()`方法，Demo类不做任何修复，`test_addMethod()`得到的结果竟然是`A a_t1`，说明`obj.a_t2`执行的是`a_t1`方法。在dex文件第一次加载的时候，会执行dexopt，dexopt有verify和optimize两个过程，optimize阶段，重写`invoke-virtual`为虚拟机内部指令`invoke-virtual-quick`，这个指令后面跟的**立即数就是该方法再`vtable`中的索引值**。

　　`invoke-virtual-quick`效率明显比`invoke-virtual`更高，直接从实际类型的vtable中获取调用方法的指针，而忽略了dvmResolveMethod从变量的引用类型获取该方法再vtable索引ID的步骤，所以更高效。

> obj.a_t2() 等价于 invoke-virtual-quick A.vtable[0]，这就是错误的原因。

***

### 3 Dalvik下完整dex方案的新探索

### 3.1 冷启动类加载修复

　　如果一个类直接引用到的所有非系统类都和该类再同一个dex中的话，那么这个类会被打上`CLASS_ISPREVERIFIED`标志。

　　类似于QFix和Tinker这类插入dex的方案，最大的问题是**如何解决Dalvik虚拟机下类的pre-verify问题**？

* QQ空间的处理方式是在每个类中插入一个来自其他dex的hack.class，由此让所有类都不满足pre-verify条件。
* Tinker的方式是合成全量的dex文件，这样所有类都在全量dex中解决，从而消除类重复而带来的冲突。
* QFix是获取虚拟机中某些底层函数，提前解析到所有补丁类。以此绕过pre-verify检查。

　　**dex比较的最佳粒度，应该是类的维度**。

### 3.2 一种新的全量Dex方案

* 传统方案：完整的dex就是把原有的dex和补丁包里的dex重新合并成一个。
* Sophix方案：原有基线包的dex中，去掉补丁中也有的类，补丁 + 去除补丁类的基线包，就是新App中的所有类了。

　　和Android原生的multi-dex同理，基线包在去掉了补丁中的类后，原有需要发生变更的类就消除了，基线包dex里就只包含不变的类了。而这些不变的类在要用到补丁中的新类时会自动去找补丁dex，补丁包中的新类在需要用到不变的类是也会找到基线包dex的类，这样基线包里面不使用补丁类的类仍旧可以按照原有的逻辑odex，最大程度保证了dexopt的效果。优点如下：

* 不需要像传统合成思路那样判断类的增加和修改情况
* 也不需要处理合成时方法数超了的情况
* 对于dex的结构也不用进行破坏性重构。

　　现在，合成完整dex的问题就简化为如何在基线包里面去掉补丁包中包含的所有类。[dex文件中的header结构和各个属性](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book2/source_code/DexHeader.md)。既然要去除dex中的类，最关心的应该是`class_defs`属性。

　　我们并不需要把某个类的所有信息都从dex移出，仅仅是使得在解析这个dex的时候找不到这个类的定义就可以了。因此，**只要移除定义的入口，对于类的具体内容不进行删除，这样就可以最大限度的减少`offset`的修改**。

* [如何去除dex中的类定义](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book2/source_code/DexClassDef.md)

### 3.3 对于Application的处理

　　Application是整个App的入口，在进入到替换的完整dex之前，一定会通过Application的代码，然而，Application必然是在加载在原来的dex里面的。只有在补丁加载后使用的类，会在新的完整的dex里找到。所以，在补丁加载后，如果Application类使用到其他新dex里的类，如果Application被打上pre-verify标志，就会抛出异常。解决办法就是清除掉标志。

类的标志位于[ClassObject](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book2/source_code/ClassObject.md)的`accessFlags`成员中。

而pre-verify标志的定义是：

```
CLASS_ISPREVERIFIED = (1<<16) // class has been pre-verified
```

因此我们只需要在JNI层清除掉它即可：

```
classObj->acccessFlags &= ~CLASS_ISPREVERIFIED
```

这样，在`dvmResolveClass`中找到新dex里的类后，由于CLASS_ISPREVERIFIED标志被清空，就不会判断所在dex是否相同，从而避免抛出异常。

* **Tinker的方案**：将自己的Appliction替换成`TinkerApplication`，并在初始化`TinkerApplication`时作为参数传入。在生命周期回调时通过反射的方式调用实际Application的相关回调逻辑。
* **Amigo方案**：在编译过程中，自定义gradle插件将App的Appliction替换成Amigo自己的另一个Appliction，并将原来的Application的name保存起来，该修复的问题都修复完再调用之前保存的Application的`attach(Context)`，然后将它回调到loadedApk中，最后调用它的`onCreate()`，执行原有Application的逻辑。

### 3.4 dvmOptResolveClass问题与对策

　　清除标志方案中，如果入口Application是没有`pre-verified`的，反而有更大问题。

　　Dalvik虚拟机如果发现某个类没有pre-verified，就会在初始化这个类时做Verify操作，将会扫描这个类的所有代码，在扫描过程中**对这个类代码里使用到的类**都进行`dvmOptResolveClass`操作。它会在解析的时候对使用到的类进行初始化，而这个逻辑是发生在Application类初始化的时候。此时补丁还没进行加载，所以就会提前加载原始dex中的类。接下来当补丁加载完毕后，当这些已经加载的类用到新dex中的类，并且又是pre-verified时就会报错。

　　这里最大的问题是*无法把补丁加载提前到dvmOptResolveClass之前，因为在一个App的生命周期里，没有可能到达比入口Application初始化更早的时期了*。问题常见于多dex，因为无法保证Application用到的类和它处于同个dex中。多dex情况下想要解决这个问题，有两种办法：

* 让Application用到的所有非系统类都和Application位于同一个dex里，这样就可以保证pre-verified标志被打打上，避免进入dvmOptResolveClass，而在补丁加载完之后，我们再清除pre-verified标志，使得接下来使用其他类也不会报错。
* 把Application里面除了热修复框架代码以外的其他代码都剥离开，单独提出放到一个其他类里面，这样使得Application不会直接用到过多非系统类，这样，保证这个单独拿出来的类和Application处于同一个dex的该类还是比较大的。如果想要更保险，Application可以采用反射方式访问这个单独类，这样就彻底把Application和其他类隔绝开了。

　　第一种方法实现简单，因为Android官方multi-dex机制会自动将Application用到的类打包到主dex中，所以只要把热修复初始化放在attachBaseContext最前面就可以了。第二种方法繁琐，是在代码架构层面进行重新设置，但是可以一劳永逸的解决问题。

***

### 4 入口类与初始化时机的选择

### 4.1 初始化时机

　　冷启动完整修复方案，本质就是换掉整个原有的dex文件。但是热修复的初始化本身也是一段代码。必须调用到这段代码，热修复操作才能执行完整。因为调用到热修复的类，肯定是使用者自己的类，这个类是无法被修复的，并且只能存在于原始安装包的classed.dex中。如果要使热修复类之前使用的其他类最少，只能放在Application类入口中。

　　真实的启动顺序是按照如下顺序进行的：

* Application.attachBaseContext 
* ContentProvider.onCreate
* Application.onCreate
* Activity.onCreate

　　所以热修复放在attachBaseContext里面最好，但是也有很多现在，此时App申请的权限还没授予完成，所以会遇到无法访问网络之类的问题。因此，可以执行初始化，但是不能进行网络请求下载补丁。

### 4.2 防不胜防的细节错误

在进行初始化的时候，经常容易错误的提早引入其他类。

```java
public class SampleApplication extends Application {
    LocalStorageUtil localStorageUtil = new LocalStorageUtil();

    protected void attachBaseContext(Context base) {
        CrashReport.initCrashReport(this);
        SophixWrApper.init(this);
        MultiDex.install(this);
        localStorageUtil.init(this);
    }
    
    public void onCreate(){
        super.onCreate();
        SophixWrApper.query();
    }
    
    public LocalStorageUtil getLocalStorageUtil() {
        return localStorageUtil;
    }
    
    static private class SophixWrApper {
        static void init(Application context){
            final SophixManager instance = SophixManager.getInstance();
            instance.setContext(context)
                    .setAppVersion(BuildConfig.VERSION_NAME)
                    .setPatchLoadStatusStub(new PatchLoadStatusListener() {
                        @Override
                        public void onLoad(final int mode,
                                           final int code,
                                           final String info,
                                           final int handlePatchVersion){
                            if(code == PatchStatus.CODE_LOAD_SUCCESS){
                                MyLogger.i("", "");
                            } else if (code == PatchStatus.CODE_LOAD_RELAUNCH){
                                MyLogger.i("", "");
                            }
                        }
                    });
            instance.initialize();
        }
        static void query() {
            SophixManager.getInstance().queryAndLoadNewPatch();
        }
    }
}
```

上面代码容易出现的几个问题：

* CrashReport.initCrashReport(this)在Sophix热修复初始化之前提早引入，必然是不行的。
* 虽然初始化确实是在attachBaseContext里，但是包装了一个SophixWrApper类，这会导致初始化之前提前引入类。因此初始化不可以包装在其他类中。
* 在setAppVersion的时候使用了BuildConfig类，这个BuildConfig类是Android编译期间动态生成的，也属于非系统类，如果这里使用就会提前加载。建议用PackageManager来获取版本号。
* 在回调类中使用了MyLogger，在回调状态的时候引用很可能热修复还未初始化完毕，因此需要换为系统类android.utils.log。
* LocalStorageUtil直接在声明处赋值了它的实例，这个赋值起始是隐式发生在对象构造函数中的，这个时候甚至早于attachBaseContext的，因此需要在初始化之后才能赋值。
* MultiDex.install(this)调用放在了热修复初始化后，这样做虽然没有引入类的问题，但是可能导致后面热修复框架初始化的时候找不到其他不在主dex中的热修复框架内部类，因此需要把它提前到热修复初始化之前。而提早引入MultiDex类不会带来问题，因为在热修复初始化之后，再也没有调用这个MultiDex类的地方。
* super.attachBaseContext(base)必须加上，否则无法正常运行。

### 4.3 入口类带来的修复限制

　　如果修改了入口Application中直接使用的类的结果，可能会引起错位异常。如果某个类的某个方法，是根据方法索引来取得的，这时候类里面新增或减少方法，可能会导致索引错位。

　　保证初始化在单独的入口类中进行，后面在用反射的方式替换为原有的Application。上面的代码案例中，可以把Sophix初始化相关逻辑移除，把初始化方法到了一个单独的SophixStubApplication类种，这个类作为AndroidManifest的入口替换掉原来的SampleApplicaiton类。

```java
public class SophixStubApplication extends SophixApplication {
    // 此处SophixEntry应指定真正的Application，并且保证RealApplicationStub类名不被混淆。
    @Keep
    @SophixEntry(MyApplication.class)
    static class RealApplicationStub {}

    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
        initSophix(this);
    }

    private void initSophix(Application context){
        String AppVersion = "0";
        try {
            AppVersion = this.getPackageManager()
                    .getPackageInfo(this.getPackageName(), 0)
                    .versionName;
        } catch (Exception e){}
        final SophixManager instance = SophixManager.getInstance();
        instance.setContext(context)
                .setAppVersion(AppVersion)
                .setAesKey(null)
                .setEnableDebug(false)
                .setEnableFullLog()
                .setPatchLoadStatusStub(new PatchLoadStatusListener() {
                    @Override
                    public void onLoad(final int mode,
                                       final int code,
                                       final String info,
                                       final int handlePatchVersion){
                        if(code == PatchStatus.CODE_LOAD_SUCCESS){
                            Log.i("", "");
                        } else if (code == PatchStatus.CODE_LOAD_RELAUNCH){
                            Log.i("", "");
                        }
                    }
                });
        instance.initialize();
    }
}
```

开发者使用这种方式的进行初始化的时候：

* 赋值这个SophixStubApplicaiton类到自己的项目中
* 把AndroidManifest里面的Application指定为它
* 设置SophixEntry为SampleApplication。

Sophix运行步骤：

* 先执行初始化逻辑
* 初始化完成后，通过反射得到SophixStubApplication的静态内部类RealApplicationStub
* 通过它的类注解SophixEntry得到真正的Application
* 然后调用SamepleApplication的生命周期函数attachBaseContext、onCreate等，再进行替换

其他热修复方案也有很多采用替换Application的方式，但是它们主要实现方式是在编译期间，通过gradle插件偷偷的把AndriodManifest中的入口Application替换掉。

不侵入编译流程可以带来巨大好处：

* 开发者在编译期间可以直接使用Android原生插件或自定义插件，不限于任何IDE和打包工具。
* 当Android原生编译链升级时，不必对新版本gradle进行适配，也不会受到其他JVM平台新语言的加入（kotlin）的影响。
* 直接作用于APK产物，稳定，无缝兼容。

***

**冷启动修复主要注意避免提前引入非系统API类**。