## 第2章 热替换代码修复

### 1.底层热替换原理

### 1.1 Andfix回顾

**Q**：为何唯独Andfix能够做到即时生效？

**A**：在App启动到一半时，所需发生变更的分类已经被加载过了，在**Android系统中时无法对一个分类进行卸载的**。而腾讯系的方案是让Classloader去加载新的类，如果不重启App，原有的类还在虚拟机中，就无法加载新类。因此，只有在下重启是，还没有运行到业务逻辑之前抢先加载补丁中的新类。

　　**Andfix是直接在已经加载的类中的native层替换掉原有方法，是在所有类的基础上进行修改**。

　　**Android 4.4以下版本用Dalvik虚拟机，4.4以上用的是Art虚拟机**。

　　**每一个Java方法在Art虚拟机中都对应一个ArtMethod，ArtMethod记录了这个Java方法的所有信息，包括所属类、访问权限、代码执行地址等**。（art/runtime/art_method.h）

　　**Andfix执行过程**：通过env->FromReflectedMethod，可以由Method对象得到这个方法所对应的ArtMethod的真正起始地址，然后强制转换成ArtMethod指针，将旧函数的所有成员变量替换为新函数的。

### 1.2 虚拟机调用方法的原理

　　Android 6.0版本中，Art虚拟机中[ArtMethod结构](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book2/art_method.h.md)中，最重要的字段是entry_point_from_interprete_和

entry_point_from_quick_compiled_code_ 了，从名字可以看出来，它们就是方法的执行入口。**Java代码在Android中会编译为Dex Code**。

　　Art虚拟机中可以采用解释模式或者AOT机器码模式执行Dex Code。

* **解释模式**，就是取出Dex Code，逐条解释执行。（entry_point_from_interpreter_）
* **AOT模式**，预先编译好Dex Code对应的机器码，然后在运行期直接执行机器码，不需要逐条解释执行Dex Code。（inter_point_from_quick_compiled_code_）

　　不论是解释模式还是AOT机器码模式，在运行期间还会需要调用ArtMethod中的其他成员字段。

　　当把一个旧方法的所有成员字段都换为新方法的成员字段后，执行时所有的数据就可以保持和新方法的数据一致。这样在所有执行到旧方法的地方，会获取到新方法的执行入口、所属类型、方法索引号以及所属dex信息，然后像调用旧方法一样执行新方法的逻辑。

### 1.3 兼容性问题的根源

　　如果某个厂商对ArtMethod结构体进行了修改，就和原有开源代码里的结构不一样，那么在这个修改过的ArtMethod结构体的设备上，替换机制就会出问题。

　　比如，在Andfix替换`declaring_class_`的地方，

```c
smeth -> declaring_class_ = dmeth -> declaring_class_;
```

　　由于`declaring_class_`是Andfix里ArtMethod的第一个成员，因此它和以下代码等价：

```c++
*(uint32_t*) (smeth + 0) = *(uint32_t*) (dmeth + 0)
```

　　由于调用变量是根据**偏移量**，第一个变量的偏移量为0，所以+0，如果某厂商在ArtMethod的结构体中的`delcaring_class_`前面添加了一个字段`additional_`，那么smeth+0的位置实际就变了，所以导致错误。

***

### 2 突破底层差异的方法

### 2.1 突破底层结构差异

　　这样的native替换思路，其实就是替换ArtMethod的所有成员。那么并不需要构造出ArtMethod具体的各个成员字段，只要把ArtMethod作为一个整体进行替换。Andfix改为如下代码：

```java
memcpy(smeth, dmeth, sizeof(ArtMethod));
```

　　这其中最关键的地方，在于sizeof(ArtMethod)的计算结果，如果计算结果有偏差，导致部分成员没被替换，或者替换区域超出了边界，都会导致严重的问题。

　　在Art中，初始化一个类的时候会给这个类的所有方法分配内存空间。根据源码分析 分配内存空间代码、函数分配方法所在区域代码、ArtMethod Array的结构，发现ArtMethod是紧密排列的，所以一个ArtMethod的大小，就是相邻两个ArtMethod的起始地址的差值。所以解决了sizeof不精确的问题。只要之后的代码，能保证ArtMethod数组仍然是线性结构排列，就无需在进行适配。

### 2.2 访问权限的问题

* **1.方法调用时的权限检查**

  　　在dex2oat生成AOT机器码时是做一些检查和优化的，由于在dex2oat编译机器码时确认了两个方法同属于一个类，所以机器码中不存在权限检查的相关代码。

* **2.同名包下的权限问题**

  　　补丁中的类在访问同包名下的类时，会报出访问权限异常`java.lang.IllegalAccessError`。因为补丁包和原有的base包的ClassLoader不是同一个，所以两个类无法被判别为同包名。校验逻辑在`Class::IsInSamePackage`中（art/runtime/mirror/class.cc）。

  　　知道了原因，只要设置新类的ClassLoader为原来类就可以了。而这一步同样不需要在JNI层构造底层的结构，只要通过反射进行设置。实现代码如下：

  ```javca
  Field classLoaderField = Class.class.getDeclareField("classLoader");
  classLoaderField.setAccessible(true);
  classLoaderField.set(newClass, oldClass.getClassLoader());
  ```

* **3.反射调用非静态方法产生的问题**

  ```java
  // BaseBug.test 方法已经被热替换了
  BaseBug bb = new BaseBug();
  Method testMeth = BaseBug.class.getDeclaredMethod("test");
  testMeth.invoke(bb);
  ```

  invoke的时候抛出异常：

  ```
  Caused by: java.lang.IllegalArgumentException: 
  	Expected receiver of type com.patch.demo.BaseBug;
  	but got com.patch.demo.BaseBug
  ```

  前者是被热替换的方法所属的类，由于我们把它的ArtMethod的delcaring_class_替换了，因此就是新的补丁类，而后者作为被调用的实例对象bb的所属类，是原有的BaseBug。两者不同。

  反射中，invoke -> InvokeMethod -> VerifyObejctsClass，VerifyObjectsClass函数中做验证，如果发现旧类和新类不匹配，就会抛出上述异常。如果是静态方法，是在类的级别直接进行调用，就不需要接收对象作为参数，也就没有这方面的检查了。

### 2.3 即时生效所带来的限制

* 直接在运行期修改底层结构的热修复方案，只能支持方法的替换。
* 补丁类中方法的增加或减少，会导致这个类的整个Dex的方法数的变化，方法数伴随着方法索引的变化，这样就无法正常的索引到正确的方法了。
* 字段的增加和减少和方法变化的情况一样。
* 新增一个完整的、原有包中不存在的是类是可以的，不受限制。

总之，只有以下两种情况不适用：

* **引起原有类中发生结构变化的修改**。
* **修复了的非静态方法会被反射调用**。

对于比较大的代码改动以及被修复方法反射调用情况，需要App冷启动。

***

### 3 编译期与语言特性的影响

### 3.1 内部类编译

**Q**：修改外部类某个方法逻辑为访问内部类的某个方法时，最后打出来的补丁包竟然提示新增一个方法？

**A**：**内部类在编译期会被编译为跟外部类一样的顶级类**。

* (1)**静态内部类/非静态内部类的区别**

  * 非静态内部类持有外部类的引用
  * 静态内部类不持有外部类的引用
  * **非静态内部类，编译期间会自动合成`this$0`域表示的就是外部类的引用**。

* (2)**内部类和外部类互相访问**

  **Q**：内部类既然是顶级类，为什么私有的method/field可以被外部类访问到？

  **A**：外部类为了访问内部类私有的域/方法，编译期间为内部类生成`access$**`数字编号相关方法。（内部类访问外部类同理）

* (3)**热部署解决方案**

  　　打补丁前未访问内部类私有域，打补丁后访问了，为了可以走热部署方案，如何避免编译器自动生成`access$**`相关方法？

  * 一个外部类如果有内部类，把所有method/field的私有访问权限改成protected或public或者默认访问权限。
  * 同时把内部类的所有method/field的私有访问权限改为protected或public或者默认访问权限。

### 3.2 匿名内部类编译

　　匿名内部类也是个内部类，但是我们发现新增一个匿名类（补丁热部署下允许新增类），同时规避上一节中的情况，但还是提示method的新增。

* (1)**匿名内部类编译命名规则**

  　　匿名内部类的名称格式一般是**外部类$数字编号**，后面的数字编号，是编译器根据匿名内部类在外部类中出现的先后关系，依次累加命名的。

  ```java
  public class DexFixDemo {
      public static void test(Context context) {
          /*new DialogInterface.OnClickListener() {
              @Override
              public void onClick(DialogInterface dialog, int which) {
                  // do something ...
              }
          };*/
          new Thread("thread-1") {
              @Override
              public void run() {
                  // do something ...
              }
          }.start();
      }
  }
  ```

  　　修复后的APK新增`DialogInterface.OnClickListener`这个匿名内部类，但是最后不定工具发现新增了`onClick`方法，因为打补丁前只有一个Thread匿名内部类，此时该类的名称是`DexFixDemo$1`，然后打补丁后在test方法中新增了`DialogInterface.OnClickListener`的匿名内部类。此时OnclickListener匿名内部类是`DexFixDemo$1`，Thread匿名内部类名称是`DexFixDemo$2`，所以此时有问题。减少一个匿名内部类同理。

* (2)**热部署解决方案**

  　　如果有补丁热部署的需求，应该极力避免插入一个新的内明内部类。如果**匿名内部类是插入到外部类的末尾，那么是允许的**。

### 3.3 有趣的域编译

* (1)**静态field，非静态field编译**

  　　**热部署方案也不支持< clinit>的修复**，这个方法会在Dalvik虚拟机中类加载的时候进行类初始化调用。Java源码中没有clinit方法，这个方法是Android编译器自动合成的。静态field的初始化喝静态代码块实际上就会被编译器编译在clinit方法中。

* (2)**静态field初始化，静态代码块**

  　　静态代码块和静态域初始化在`clinit`中的先后关系就是两者出现在源码中的先后关系。类加载进行类初始化的时候，会去掉哦那个clinit，一个类仅加载一次。

  以下三种情况会尝试去加载一个类：

  * 1.创建一个类的对象（new-instance指令）；
  * 2.调用类的静态方法（invoke-static指令）；
  * 3.获取类的静态域的值（sget指令）。

  　　首先判断这个类有没有被加载过，如果没有，执行dvmResolveClass -> dvmLinkClass -> dvmInitClasser的流程，类的初始化时在dvmInitClasser中。dvmInitClass这个函数首先会尝试对父类进行初始化，然后调用本类的clinit方法，所以此时静态field得到初始化并且静态代码块得到执行。

* (3)**非静态field初始化，非静态代码块**

  　　非静态field初始化和非静态代码块被编译器翻译在`< init>`默认无参构造函数中。**实际上如果存在有参构造函数，那么每个有参构造函数都会执行一个非静态域的初始化和非静态代码块**。

  > 构造函数会被Android编译器自动翻译成< init>方法

  　　`init`在对类对象进行初始化的时候被调用。new一个对象的时候，首先执行`new-instance`指令，主要为对象分配堆内存空间，同时如果类之前没加载过，尝试加载类，然后执行`invoke-direct`指令调用类的init构造函数方法执行对象的初始化。

### 3.4 final static域编译

　　非静态域编译到`init`方法中，静态域编译到`clinit`方法中，`final static`域是一个静态域，但是它修饰的基本类型或String常量类型，并未编译到clinit中。

* (1)**final static域编译规则**

  ```java
  public class DexFixDemo {
      static Temp t1 = new Temp(); // t1 == NULL
      final static Temp t2 = new Temp(); // t2 == NULL
  
      final static String s1 = new String("heihei"); // 引用类型 // // s1 == NULL
      final static String s2 = "haha"; // 字符串常量 // s2 == "haha"
  
      static int i1 = 1; // i1 == 0;
      final static int i2 = 2; // i2 == 2
  }
  ```

  　　从反编译得到的smali文件中发现，`i2`和`s2`两个静态域竟然没被初始化。其他的非final静态域均在clinit函数中得到初始化。类加载初始化dvmInitClass在执行clinit方法之前，首先会执行`initSFields`，这个方法的作用主要就是给static域赋予默认值。如果是引用类型，默认初始值为NULL。其中`t1/t2/s2/i1`在initSFields中赋值了默认初始化值，然后在clinit中复制了程序设置的值。而i2/s2在initSFields得到的默认值就是程序中设置的值。

  **static 和 final static修饰field的区别**，如下：

  * final static修饰的**原始类型**和**`String`类型域**（**非引用类型**），并不会被编译在clinit方法中，而是在类初始化执行initSFields方法时得到了初始化赋值。
  * final static修饰的**引用变量**，初始化仍然在clinit方法中。

* (2)**final static域优化原理**

  　　**如果一个field是常量，那么推荐尽量使用`static final`作为修饰符**。这句话仅限于修饰**原始类型**和String类型域（非引用类型）。

  ```java
  class Temp {
      public static void test() {
          int i1 = DexFixDemo.i1;
          int i2 = DexFixDemo.i2;
          Temp t1 = DexFixDemo.t1;
          Temp t2 = DexFixDemo.t2;
          String s1 = DexFixDemo.s1;
          String s2 = DexFixDemo.s2;
      }
  }
  ```

  从反编译得到的smali文件中知道：

  * Temp获取`DexFixDemo.is`（final static域）是直接通过`const/4`（获取的值是**立即数**）指令；
  * 获取`DexFixDemo.i1`（非final域），通过`sget`指令。
    * 首先判断这个域之前是否被解析过
    * 如果没有，尝试解析域
    * 如果这个静态域所在类没有被解析，先解析类
    * 然后拿到这个sfield静态域，得到这个静态域的值。

  从上可以看出`sget`指令比`const/4`指令解析过程复杂，所以**final static基本类型可以得到优化**。

  **dex文件中有一块区域存储这程序所有的字符串常量，最终这块区域会被虚拟机完整的加载到内存中，这块区域也就是通过所说的"`字符串常量区`"内存**。

  final static修饰引用类型，其作用仅仅是让编译器能在编译期间检测到final域有没有被修改。

* (3)**热部署解决方案**

  * 修改final static基本类型或String类型域（非引用类型域），由于在编译期引用到基本类型的地方被立即数替换，引用到String类型（非引用类型）的地方被常量池ID替换，所以在热部署模式下，最终所有引用到该final static域的方法都会被替换。实际上此时仍然可以执行热部署方案。
  * 修改final static引用类型域，是不允许的。因为这个field的初始化会被编译到clinit中，所以不能热部署。

### 3.5 有趣的方法编译

* (1)**应用混淆方法编译**

  项目如果应用了混淆方法编译，可能导致方法的内联和裁剪。

* (2)**方法内联**

  * 方法没有被其他任何地方调用，会被内联掉。
  * 方法足够简单，比如一个方法的实现只有一行代码，该方法会被内联掉，那么任何调用该方法的地方都会被该方法的实现替换掉。
  * 方法只被一个地方引用，这个地方会被方法的实现替换掉。

* (3)**方法裁剪**

  ```java
  class BaseBug {
      public static void test(Context context) {
          Log.d("BaseBug", "test");
      }
  }
  ```

  　　test方法中的context参数没有被使用，将会被裁剪，混淆任务首先生成裁剪后的无参方法，然后再混淆。所以补丁中修改了test方法，恰好用到了context参数，所以会检测到新增了方法。

  　　如何让参数不被裁剪？只要不让编译器在优化的时候认为引用是无用的参数就好。有效方法：

  ```java
  public static void test(Context context) {
      if (Boolean.FALSE.booleanValue()){
          context.getApplicationContext();
      }
      Log.d("BaseBug", "test");
  }
  ```

  　　**注意**：不能使用基本类型false，必须用包装类Boolean，因为如果使用基本类型if语句也可能被优化掉。

* (4)**热部署解决方案**

  混淆配置文件加上`-dontoptimize`项就不会做方法的裁剪和内联了。

  `optimization step`：进一步优化代码，非入口点的类和方法可以被设置为private、static或final，无用的参数可能被移除，并且一些方法可能会被内联。

  `preverification step`：针对`.class`文件的预校验，在`.class`文件中加上StackMa/StackMapTable信息，这样HotSpot VM在类加载时执行类校验阶段会省去一些步骤，因此类加载会更快。

  补丁热部署模式下，混淆配置最好都加上`-dontoptimize`这项。Android中混淆配置一把都需要加上`-dontpreverify`这一项。

  混淆裤对反射处理的影响：

  ```java
  Class clz = Class.forName("com.taobao.test.Temp");
  Method method = clz.getDeclaredMethod("test", new Class[]{Context.class, String.Class});
  ```

  上述代码中，com.taobao.test.Temp类不会被移除，test方法可能在shrinking阶段被移出或者在obfuscation阶段被重命名，最后导致反射失败。

  ```java
  Method method = com.taobao.test.Temp.class.getDeclaredMethod("test", 
      new Class[]{Context.class, String.Class});
  ```

  上述代码中，com.taobao.test.Temp类不会被移除，test方法不会被移出，obfuscation阶段这个方法和.class这一行代码的字节码同时发生变更，所以不会失败。

### 3.6 switch case语句编译

资源新旧替换时，存在switch case语句中的ID不会被替换掉的情况。

* (1)**switch case语句编译规则**

  * case项是连续几个比较相近的值，如1、3、5时，switch case语句编译成`packed-switch`指令。
  * case项不够连续时，如1、3、10，switch case语句被编译成`sparse-switch`指令。

  编译器会决定怎样的值是否才算连续。

* (2)**热部署解决方案**

  　　一个资源ID肯定是`const final static`变量，此时恰好switch case语句被编译成packed-switch指令，所以这时候不做任何处理就会存在资源ID替换不完全的情况。所以只要修改smali反编译流程，碰到packed-switch指令强转为sparse-switch指令。`:pswitch_N`等相关标签指令也需要强转为`:sswitch_N`指令。然后做资源ID的暴力替换，然后再回编译smali为dex。再做类方法变更的检测。**反编译-资源ID替换-回编译**。

### 2.7 泛型编译

