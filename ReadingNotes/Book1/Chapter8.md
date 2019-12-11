## 第8章 虚拟机字节码执行引擎

　　在不同的虚拟机实现里面，执行引擎在执行Java代码的时候可能会有**解释执行**（通过解释器执行）和**编译执行**（通过即时编译器产生本地代码执行）两种选择。

***

### 1. 运行时栈帧结构

　　**栈帧**用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区中的虚拟机栈的栈元素。

　　栈帧包括局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息。在编译程序代码的时候，栈帧中需要多大的局部变量表，多深的操作数栈都已经完全确定了，并且写入到方法表的Code属性中。

　　一个线程中的方法调用链可能会很长，很多方法都同时处于执行状态。对于执行引擎来说，在活动线程中，只有位于栈顶的栈帧才是有效的，称为**当前栈帧**，与这个栈帧相关联的方法称为**当前方法**。执行引擎运行的所有字节码指令都只针对当前栈帧进行操作。栈帧结构图如下：

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/stackframe.png)

### 1.1 局部变量表

　　局部变量表是一组变量值存储空间，用于存放**方法参数**和**方法内部定义的局部变量**。在Java程序编译为Class文件时，就在方法的Code属性的max_locals数据项中确定了该方法所需分配的局部变量的最大容量。局部变量表的容量以**变量槽**（Slot）为最小单位，Slot所占用的内存空间大小并未明确指明，只是每个Slot都应该能存放一个boolean、byte、char、short、int、float、reference或returnAddress类型的数据。

　　由于局部变量表建立在线程的堆栈上，是线程私有，无论读写两个连续的Slot是否为原子操作，都不会引起数据安全问题。（Java中64位的数据类型只有long和double）

　　局部变量表的Slot可以重用，这样设计除了节省栈空间，还有副作用，如影响系统的垃圾收集行为。

```
// 代码1：局部变量表Slot复用对垃圾收集的影响之一
public static void main(String[] args) {
    byte[] placeholder = new byte[64 * 1024 * 1024];
    System.gc(); // 未回收placeholder所占内存
}

// 代码2：局部变量表Slot复用对垃圾收集的影响之二
public static void main(String[] args) {
    {
        byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    System.gc(); // 未回收placeholder所占内存
}

// 代码3：局部变量表Slot复用对垃圾收集的影响之三
public static void main(String[] args) {
    {
        byte[] placeholder = new byte[64 * 1024 * 1024];
    }
    int a = 0;
    System.gc(); // 回收placeholder所占内存
}
```

placeholder能否被**回收的根本原因**是：局部变量表中的Slot是否还存有关于placeholder数组对象的引用。

* 代码1没有回收placeholder所占内存，因为在执行System.gc()时，变量placeholder还处于作用域之内，虚拟机自燃不敢回收placeholder内存。
* 代码2加入花括号，placeholder作用域被限制在花括号内，代码虽然离开了placeholder的作用域，但此之后，没有任何局部变量表的读写操作，placeholder原本占用的Slot并未被复用，所以做为GC Roots一部分的局部变量表仍然保持着对它的关联。
* 如果遇到一个方法，其后面的代码有一些耗时很长的操作，而前面又定义了占用了大量内存、实际上已经不会再使用的变量，手动将其设置为null值（用来代替那句int a = 0，把变量对应的局部变量表Slot清空）便不见得是一个绝对不无意义的操作（书《Practical Java》中把"不使用的对象手动赋值为null"作为一条推荐的编码规则）。赋null值的操作在经过JIT编译优化后就会被消除掉，此时便没有意义。

**注：局部变量定义了但没有赋初始值是不能使用的。**

### 1.2 操作数栈

　　操作数栈也称为操作栈，后入先出。操作数栈的最大深度为Code属性的max_stacks数据项的值。

　　当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中、会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈/入栈操作。例如，在做算数运算的时候是通过操作数栈来进行的，又或者在调用其他方法的时候是通过操作数栈来进行参数传递的。

　　**eg**：整数加法的字节码指令iadd在运行的时候操作数栈中最接近栈顶的两个元素已经存入了两个int型的数值，当执行这个指令时，会将这两个int值出栈并相加，然后将相加的结果入栈。

### 1.3 动态连接

　　每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有这个引用是为了支持方法调用过程的动态连接。Class文件的常量池中存在大量的符号引用，字节码中的方法调用指令就以常量池中指向方法的符号引用作为参数。这些符号引用一部分会在类加载阶段或者第一次使用的时候就转化为直接引用，这种转化称为**静态解析**。另外一部分将在每一次运行期间转化为直接引用，这部分称为**动态连接**。

### 1.4 方法返回地址

当一个方法开始执行后，只有两种方式可以退出这个方法：

* **正常完成出口**：执行引擎遇到任意一个方法返回的字节码指令。
* **异常完成出口**：方法执行过程遇到异常，并且这个异常没有在方法体内得到处理。

方法退出的过程实际上是把当前栈帧出栈，因此退出时可能执行的操作有：恢复上层方法的局部变量表和操作数栈，把返回值压入调用者栈帧的操作数栈中，调用PC计数器的值以指向方法调用指令后面的一条指令等。

***

### 2. 方法调用

　　方法调用并不等同于方法执行，方法调用阶段唯一的任务就是确定被调用方法的版本。Class文件的编译过程中不包含传统编译中的链接步骤，一切方法调用在Class文件里面存储的都只是符号引用，而不是方法在实际运行时内存布局中的入口地址（直接引用）。

### 2.1 解析

　　所有方法调用中的目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这种解析能成立的前提是：方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期时不可改变的。这类方法的调用称为**解析**。

　　Java虚拟机里面提供了5条调用字节指令，如下：

* **invokestatic**：调用静态方法
* **invokevirtual**：调用所有虚方法
* **invokespecial**：调用实例构造器< init>方法、私有方法和父类方法
* **invokeinterface**：调用接口方法，会在运行时再确定一个实现此接口的对象
* **invokedynamic**：现在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。

　　只要能被invokestatic和invokespecial指令调用的方法，都能在解析阶段中确定位移调用版本，符合这个条件的有静态方法、私有方法、实例构造器、父类方法4类，它们在类加载的时候就会把符号引用解析为该方法的直接引用，这些方法称为**非虚方法**，与之相反，其他的方法称为**虚方法**（除去final方法）。final方法虽然使用invokevirtual指令来调用，但是由于无法覆盖，所以**final方法是一种非虚方法**。

### 2.2 分派

* 1.**静态分派**　　

  * 方法静态分派演示，代码如下：

    ```java
    // 方法静态分派演示
    public class StaticDispatch {
        static abstract class Human { }
        static class Man extends Human { }
        static class Woman extends Human { }
        public void sayHello(Human guy) {
            System.out.println("hello guy!");
        }
        public void sayHello(Man guy) {
            System.out.println("hello gentleman!");
        }
        public void sayHello(Woman guy) {
            System.out.println("hello lady!");
        }
        public static void main(String[] args) {
            Human man = new Man();
            Human woman = new Woman();
            StaticDispatch sr = new StaticDispatch();
            sr.sayHello(man);   // 运行结果：hello guy!
            sr.sayHello(woman); // 运行结果：hello guy!
        }
    }
    ```

    上述代码结果选择执行参数类型为Human的重载。

    ```java
    Human man = new Man();
    ```

    上面的代码，"Human"称为变量的**静态类型**，或叫做外观类型；"Man"称为变量的**实际类型**。静态类型的变化仅仅在使用时发生，变量本身的静态类型不会改变，最终的静态类型编译器可知；实际类型的变化结果运行期才确定。例如下面的代码：

    ```java
    // 实际类型变化
    Human man = new Man();
    man = new Woman();
    // 静态类型变化
    sr.sayHello((Man) man);
    sr.sayHello((Woman) man);
    ```

    ***虚拟机（准确的说是编译器）在重载时是通过参数的静态类型而不是实际类型作为判定依据的。***

    所有依赖静态类型来定位方法执行版本的分派动作称为**静态分派**。静态分派典型的应用就是方法重载。静态分派发生在*编译阶段*，因此确定静态分派的动作实际上不是由虚拟机来执行的。

  * 重载方法匹配优先级

    ```java
    public class Overload {
        public static void sayHello(Object arg) { // 6
            System.out.println("hello Object");
        }
        public static void sayHello(int arg) { // 2
            System.out.println("hello int");
        }
        public static void sayHello(long arg) { // 3
            System.out.println("hello long");
        }
        public static void sayHello(Character arg) { // 4
            System.out.println("hello Character");
        }
        public static void sayHello(char arg) { // 1
            System.out.println("hello char");
        }
        public static void sayHello(char... arg) { // 7
            System.out.println("hello char ...");
        }
        public static void sayHello(Serializable arg) { // 5
            System.out.println("hello Serializable");
        }
        public static void main(String[] args) {
            sayHello('a');
        }
    }
    ```

    代码中方法后的数字，代表了重载优先级。

    * hello char：上面代码首先输出的是：hello char。，因为'a'是char类型。
    * hello int：'a'除了可以代表字符串，还可以代表数字97（Unicode数值）
    * hello long：97进一步转型为长整数97L。（char->int->long->float->double）
    * hello Character：自动装箱，'a'被包装为它的封装类型java.lang.Character
    * hello Serializable：java.lang.Serializable是java.lang.Character类实现的一个接口。char可以转型为int，但是Character不能转型为Integer，它只能转型为它实现的接口和父类。
    * hello Object：char装箱后转型父类，有多个父类，将在继承关系中从下往上搜索。
    * hello char ...：变长参数重载优先级最低。此时'a'被当做一个数组元素。在单个参数中能成立的自动转型，如char转型为int，在变长参数中是不成立的。

    静态方法会在类加载器就进行解析，而静态方法也是可以拥有重载版本的，选择重载版本的过程也是通过静态分派完成的。

* 2.**动态分派**

  invoke执行的运行时解析过程大致分为以下几步：

  * 1）找到操作数栈顶的第一个元素所指向的对象的实际类型，记做C。
  * 2）如果在类型C中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，如果通过则返回这个方法的引用，查找过程结束；如果不通过，则返回java.lang.IllegalAccessError异常。
  * 3）否则，按照继承关系从下往上依次对C的各个父类进行第2步的搜索和验证过程。
  * 4）如果始终没找到何时的方法，则抛出java.lang.AbstractMethodError异常。

  　　由于invokevirtual指令执行的第一步就是在运行期确定接收者的实际类型，所以两次调用中的invokevirtual指令把常量池中的类方法符号引用解析到了不同的直接引用上，这个过程就是Java语言中方法重写的本质。我们把这种运行期根据实际类型确定方法执行版本的分派过程称为**动态分派**。

* 3.**单分派和多分派**

  　　方法的接收者与方法的参数统称为方法的**宗量**。单分派是根据一个宗量对目标方法进行选择，多分派是根据多于一个宗量对目标方法进行选择。

  　　编译阶段编译器的选择过程，也就是静态分派的过程。静态分派属于多分派类型。

  　　运行阶段虚拟机的选择，也就是动态分派的过程。动态分派属于单分派类型。

* 4.**虚拟机动态分派的实现**

  　　由于动态分派非常频繁，为考虑性能问题，常用优化手段是为类在方法区中建立一个**虚方法表**（vtable，与此对应，在invokeinterface执行时会有接口方法表，itable），使用虚方法表索引来代替元数据查找。（优化手段除了虚方法表，还会使用内联缓存和守护内联两种手段）

  　　虚方法表中存放着各个方法的实际入口地址。如果某个方法在子类中没有被重写，那么子类的虚方法表里面的地址入口是父类的相同方法的地址入口。如果子类重写了这个方法，子类方法表中的地址入口是子类实现版本的入口地址。

  　　方法表一般在类加载的连接阶段进行初始化，准备了类的变量初始值后，虚拟机会把该类的方法表也初始化完毕。

### 2.3 动态类型语言支持

　　invokedynamic指令是JDK7 实现"动态类型语言"支持而进行的改进之一。

* 1.**动态类型语言**

  　　动态类型语言的关键特征是它的类型检查的主体过程是在运行期而不是编译期，编译期就进行类型检查过程的语言（如C++和Java等）就是最常用的静态类型语言。

  　　举个例子来解释"类型检查"，如下代码：

  ```java
  obj.println("hello world"); // 这一行代码"没头没尾"是无法执行的
  ```

  　假设这行代码在Java中，且变量obj的静态类型是java.io.PrintStream，那么obj的实际类型必须是PrintStream的子类才是合法的。否则，即使obj所属的类有println(String)方法，但与PrintStream接口没有任何继承关系，代码依旧类型检查不合法。（ECMAScript中，无论obj是何种类型，只要类型的定义中包含println(String)方法，那就可以调用成功）

  　　因为Java语言在编译期已将println(String)方法完整的符号引用（CONSTANT_InterfaceMethodref_info常量）生成出来，做为方法调用指令的参数存储到Class文件中，如下代码：

  ```
  invokevirtual #4; //Method java/io/PrintStream.println:(Ljava/lang/String;)V
  ```

  　　这个符号引用包含了此方法定义在哪个具体类型之中、方法的名字以及参数顺序、参数类型和方法返回值信息，通过这个引用，虚拟机可以翻译出这个方法的直接引用。

  　　"变量无类型而变量值才有类型"这个特点也是动态类型语言的一个重要特征。

* 2.**java.lang.invoke包**

  　　这个包的主要目的是在之前单纯依靠符号引用来确定调用的目标方法这种方式以外，提供一种新的动态确定目标方法的机制，称为**MethodHandle**（类似于C/C++中的Function Pointer，或者C#的Delegate。将可以将函数做为参数传递）。

  　　**MehodHandle演示代码**如下：

  ```java
  import java.lang.invoke.MethodHandle;
  import java.lang.invoke.MethodHandles;
  import java.lang.invoke.MethodType;
  
  public class MethodHandleTest {
      static class ClassA {
          public void println(String s) {
              System.out.println(s);
          }
      }
      public static void main(String[] args) throws Throwable {
          Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
          // 无论obj最终是哪个实现类，下面这局都能正确调用到println方法
          getPrintlnMH(obj).invokeExact("icyfenix");
      }
      private static MethodHandle getPrintlnMH(Object receiver) throws Throwable {
          // MethodType：代表"方法类型"，methodType(方法的返回值, 具体参数...);
          MethodType mt = MethodType.methodType(void.class, String.class);
          // lookup()作用是指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄
          // 因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，
          // 也就是this指向的对象，这个参数以前是放在参数列表中进行传递的，现在由bindTo来完成
          return MethodHandles.lookup().findVirtual(receiver.getClass(), "println", mt)
                  .bindTo(receiver);
      }
  }
  ```

  　　方法getPrintlnMH中模拟了invokevirtual指令的执行过程。这个方法的返回值（MethodHandle对象）可以视为对最终调用方法的一个引用。调用方法可以用如下代码：

  ```java
  void sort(List list, MethodHandle compare)
  ```

  MethodHandle的使用方法和效果与Reflection有众多相似之处，它们区别如下：

  * Reflection和MethodHandle都是在模拟方法调用，但是Reflection是模拟代码层次的方法调用，MethodHandle是模拟字节码层次的方法调用。MethodHandle.lookup中的3个方法——findStatic()、findVirtual()、findSpecial()对应invokestatic、invokevirtual&invokeinterface、invokesepcial这几条字节码指令的执行权限校验行为。
  * Reflection中的java.lang.reflect.Method对象远比MethodHandle中的java.lang.invoke.Methodhandle对象所包含的信息多。前者是方法在Java段的全面映像，包含了方法的签名、描述符、以及方法属性表中各种属性的java端标识方式，还包含运行执行权限等运行期信息。而后者仅仅包含与执行该方法相关的信息。Reflection是重量级，MethodHandle是轻量级。

* 3.**invokedynamic指令**

  　　某种程度上，invokedynamic指令与MethodHandle机制的作用是一样的。都是为了解决原有4条"invoke"指令方法分派固化在虚拟机之中的问题。

  　　**每一处含有invokedynamic指令的位置都称作"动态调用点"**。这条指令的第一个参数不再是代表方法符号引用的CONSTANT_Methodref_info常量，而变为CONSTANT_InvokeDynamic_info常量。这个新常量中可以得到3项信息：引导方法、方法类型和名称。引导方法有固定参数，并且返回值是java.lang.invoke.CallSite对象，这个代表真正要执行的目标方法调用。

  　　**invokedynamic指令演示**如下：

  ```java
  import java.lang.invoke.CallSite;
  import java.lang.invoke.ConstantCallSite;
  import java.lang.invoke.MethodHandle;
  import java.lang.invoke.MethodHandles;
  import java.lang.invoke.MethodType;
  
  public class InvokeDynamicTest {
      public static void main(String[] args) throws Throwable {
          INDY_BootstrapMethod().invokeExact("icyfenix");
      }
      public static void testMethod(String s) {
          System.out.println("hellot String : " + s);
      }
      public static CallSite BootstrapMethod(MethodHandles.Lookup lookup,
                                             String name, MethodType mt) throws Throwable {
          return new ConstantCallSite(lookup.findStatic(InvokeDynamicTest.class, name, mt));
      }
      private static MethodType MT_BootstrapMethod() {
          return MethodType.fromMethodDescriptorString(
                  "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/" +
                          "invoke/MethodType;)Ljava/lang/invoke/CallSite;", null);
      }
      private static MethodHandle MH_BootstrapMethod() throws Throwable {
          return MethodHandles.lookup().findStatic(InvokeDynamicTest.class,
                  "BootstrapMethod", MT_BootstrapMethod());
      }
      private static MethodHandle INDY_BootstrapMethod() throws Throwable {
          CallSite cs = MH_BootstrapMethod().invokeWithArguments(MethodHandles.lookup(), "testMethod",
                  MethodType.fromMethodDescriptorString("(Ljava/lang/String;)V", null));
          return cs.dynamicInvoker();
      }
  }
  ```

  上述代码使用了一个把字节码转换为invokedynamic的简单工具**INDY**来完成，所以代码中的方法名称不能随意改动，更不能把几个方法合并到一起写，因为它们是要被INDY工具读取的。

* 4.**掌控方法分派规则**

  invokedynamic指令与前面4条"invoke"指令的最大差别就是它的分派逻辑不是由虚拟机来决定的，而是由程序员决定的。

  ```java
  public class Test {
      class GrandFather {
          void thinking() {
              System.out.println("i am grandfather");
          }
      }
      class Father extends GrandFather {
          void thinking() {
              System.out.println("i am father");
          }
      }
      class Son extends Father {
          void thinking() {
              try {
                  MethodType mt = MethodType.methodType(void.class);
                  MethodHandle mh = MethodHandles.lookup().findSpecial(GrandFather.class,
                          "thinking", mt, getClass());
              } catch (Throwable e) {
                  e.printStackTrace();
              }
          }
      }
      public static void main() {
          (new Test().new Son()).thinking(); // 打印结果为：i am grandfather
      }
  }
  ```

  使用纯粹的Java语言，在Son类中是无法获取一个实际类型是GrandFather的对象引用，而invokevirtual指令的分派逻辑就是按照方法接收者的实际类型进行分派，这个逻辑是固化在虚拟机中，程序员无法改变。但是如上述代码，可以通过MehtodHandle来解决问题。

