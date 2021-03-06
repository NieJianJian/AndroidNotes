## 第7章 虚拟机类加载机制

　　虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的**类加载机制**。

　　在Java语言里面里，类型的加载、连接和初始化过程都是在程序运行期间完成的。Java天生可以动态扩展的语言特性就是依赖运行期动态加载连接这个特点实现的。例如：

* 如果编写一个面向接口的应用程序，可以等到运行时再指定其实际的实现类；
* 用户可以通过Java预定义的和自定义类加载器，让一个本地的应用程序可以在运行时从网络或其他地方加载一个二进制流作为程序代码的一部分。

***

### 1.类加载的时机

　　类从被加载带虚拟机内存开始，到卸载出内存为止，它的整个生命周期包括：加载、验证、准备、解析、初始化、使用和卸载7个阶段。其中验证、准备、解析3个部分统称为连接。如下图：

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/classlifecycle.jpg)

* 加载、验证、准备、初始化和卸载这5个阶段顺序是确定的，类加载过程必须按照这个顺序开始。

* 解析阶段不固定：它在某些情况下可以在初始化之后，为了运行时绑定（也称为动态绑定）。

* 虚拟机严格规定了**有且只有**5种情况必须立即对类进行"初始化"(而加载、验证、准备自然在此之前)：

  * 1）遇到new、getstatic、putstatic或invokestatic这4条字节码指令时，如果类没有进行过初始化，需要先触发其初始化。常见场景有：使用new实例化对象、读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。
  * 2）使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有初始化，需要先触发其初始化。
  * 3）初始化一个类时，如果其父类还没初始化，需要先触发其父类的初始化。
  * 4）当虚拟机启动时，用户需要指定一个要执行的主类（包含main()方法），虚拟机会先初始化这个类。
  * 5）当使用JDK1.7的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解析结果REF_getstatic、REF_putstatic、REF_invokeStatic的方法句柄，并且这个方法句柄所对应的类没有进行初始化，需要先触发其初始化。

  这5种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用。下面举3个例子说明何为被动引用。

  * 案例一：**通过子类引用父类的静态字段，不会导致子类初始化**。

  ```java
  package org.fenixsoft.classloading;
  public class SuperClass {
      static {
          System.out.println("SuperClass init!");
      }
      public static int value = 123;
  }
  public class SubClass extends SuperClass{
      static {
          System.out.println("SubClass init!");
      }
  }
  public class NoInitialization {
      public static void main(String[] args) {
          System.out.println(SubClass.value);
      }
  }
  ```

  输出结果"SuperClass init!"，对于静态字段，只有直接定义这个字段的类才会被初始化。因此通过其子类来引用父类中定义的静态字段，只会触发父类的初始化而不会触发子类的初始化。

  * 案例二：**通过数组定义来引用类，不会触发此类的初始化**。

  ```java
  package org.fenixsoft.classloading;
  public class NotInitialization {
      public static void main(String[] args) {
          SuperClass[] sca = new SuperClass[10];
      }
  }
  ```

  运行后并未输出"SuperClass init!"，说明并未触发SuperClass初始化。但是这段代码触发了另一个名为"[Lorg.fenixsoft.classloading.SuperClass"的类的初始化，对于用户代码带来，这并不是一个合法的的类的名称，它是一个由虚拟机自动生成的、直接继承于java.lang.Object的子类，创建由字节码指令newarray触发。这个类代表了一个元素类型为org.fenixsoft.classloading.SuperClass的一堆数组，数组中应有的属性和方法（用户可直接使用的只有被修饰为public的length属性和clone()方法）都实现在这个类里。Java语言中对数组的访问比C/C++相对安全是因为这个类封装了数组元素的访问方法（越界检查不是封装在数组元素访问的类中，而是封装在数组访问的xaload、xastore字节码指令中），而C/C++直接翻译为对数组指针的移动。

  * 案例三：**常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。**

  ```java
  public class ConstClass {
      static {
          System.out.println("ConstClass init!");
      }
      public static final String HELLOWORLD = "hello world";
  }
  public class NotInitialization {
      public static void main(String[] args) {
          System.out.println(ConstClass.HELLOWORLD);
      }
  }
  ```

  运行后并未输出"ConstClass init!"，是因为虽然在Java源码中引用了ConstClass类中的常量HELLOWORLD，但其实在编译阶段通过常量传播优化，已经将此常量值存储到了NotInitialization类的常量池中，以后NotInitialization对常量ConstClass.HELLOWORLD的引用实际都被转换为NotInitialization类对自身常量池的引用了。

　　接口中不能使用static{}语句块，但编译器仍然会为接口生成"< clinit>()"类构造器，用于初始化接口中所定义的成员变量。**接口在初始化时，只有在真正是用到父类的时候，父类才会初始化。**

***

### 2.类加载过程

### 2.1 加载

加载阶段，虚拟机完成以下3件事：

* 1）通过一个类的全限定名来获取定义此类的二进制字节流。
* 2）将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
* 3）在内存中生成一个代表这个类的java.lang.Class对象，做为方法区这个类的各种数据的访问人口。

"通过一个类的全限定名来获取定义此类的二进制字节流"并未指明二进制字节流要从哪里获取、怎么获取。建立在此基础上的一些Java技术，如：

* 从ZIP包中读取，这很常见，最终成为日后JAR、EAR、WAR格式的基础。
* 从网络中获取，这种场景最典型的应用就是Applet。
* 运行时计算生成，这种场景使用最多的就是动态代理技术，在java.lang.reflect.Proxy中，就是用了ProxyGenerator.generateProxyClass来为特定接口生成形式为"*$Proxy"的代理类的二进制字节流。
* 由其他文件生成，典型场景是JSP应用，即由JSP文件生成对应的Class类。
* 从数据中读取，这种场景比较少见，例如有些中间件服务器（如SAP Netweaver）可以选择把程序安装到数据库中来完成程序代码在集群间的分发。

**非数组类**：可以通过定义自己的类加载器去控制字节流的获取方式（即重写一个类加载器的loadClass()方法）。

**数组类**本身不通过类加载器创建，它是由Java虚拟机直接创建。但数组类的元素类型（指的是数组去掉所有维度的类型）最终是要靠类加载器去创建。

一个数组类创建过程遵循以下规则：

* 如果数组的组件类型（数组去掉一个维度的类型）是引用类型，那就递归采用本节中定义的加载过程去加载这个组件类型，数组C将在加载该组件类型的类加载器的类名称空间上被标识（**一个类必须与类加载器一起确定唯一性**）。
* 如果数组的组件类型不是引用类型（如int[]数组），Java虚拟机将会把数组C标记为与引导类加载器关联。
* 数组类的可见性与它的组件类型的可见性一致，如果组件类型不是引用类型，那数组类的可见性将默认为public。

　　加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，方法区中的数据存储格式由虚拟机实现自行定义，虚拟机规范未规定此区域的具体数据结构，然后在内存中实例化一个java.lang.Class类的对象（并没有规定是在Java堆中，对于HotSpot虚拟机而言，Class对象比较特殊，它虽然是对象，但存放在方法区里面），这个对象将做为程序访问方法区中的这些类型数据的外部接口。

### 2.2 验证

　　**目的**：为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。如果验证到输入的字节流不符号Class文件格式的约束，虚拟机会抛出java.lang.VerifyError异常或其子类异常。

* 1.**文件格式验证**

  * 是否以魔数0xCAFEBABE开头。
  * 主、次版本号是否在当前虚拟机处理范围之内。
  * 常量池的常量中是否有不被支持的常量类型（检查常量tag标志）。
  * 指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量。
  * CONSTANT_Utf0_info型的常量中是否有不符合UTF8编码的数据。
  * Class文件中各个部分及文件本身是否有被删除的或附加的其他信息。

  该验证阶段的**目的**是保证输入的字节流能正确地解析并存储于方法区之内。这阶段基于二进制字节流进行，通过验证后才会进入内存的方法区进行存储。后3个验证阶段全部是基于方法区地存储结构进行的，不会再直接操作字节码。

* 2.**元数据验证**

  * 这个类是否有父类（除了java.lang.Object之外，所有的类都应当有父类）
  * 这个类非父类是否继承了不允许被继承的类（被final修饰的类）
  * 如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法。
  * 类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，如果方法参数一致，但返回值类型却不同等）。

  该阶段**目的**是对类的元数据信息进行语义校验。

* 3.**字节码验证**

  这个阶段对类的方法体进行校验分析，保证被校验类的方法在运行时不会做出危害虚拟机安全的事件。

  * 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似的情况：在操作栈放置了一个int类型的数据，使用时却按long类型加载入本地变量表。
  * 保证跳转指令不会跳转到方法体以外的字节码指令上。
  * 保证方法体中的类型转换是有效的，例如可以把一个子类对象复制给父类数据类型，这是安全的，但是把父类对象赋值给子类数据类型，甚至把对象复制给与它毫无继承关系、完全不相干的一个数据类型。

* 4.**符号引用验证**

  此阶段发生在虚拟机将符号引用转化为直接引用的时候，这个转化动作将发生在连接的第三阶段——解析阶段发生。符号引用验证可以看作是对类自身以外（常量池中的各种符号引用）的信息进行匹配性校验。

  * 符号引用中通过字符串描述的全限定名是否能找到对应的类。
  * 在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段。
  * 符号引用中的类、字段、方法的访问性是否可被当前类访问。

### 2.3 准备

　　**目的**：为类变量分配内存并设置类变量初始值（初始值"通常情况"下为数据类型的零值），这些变量所使用的内存都将在方法区中进行分配。这时候进行内存分配的仅包括类变量（static），实例变量将在对象实例化时随着对象一起分配在Java堆中。

```java
public static int value = 123;
```

　　变量value在准备阶段后的初始值为0而不是123，而把value赋值为123的putstatic指令是程序被编译后，存放在类构造器< clinit>()方法中，所以把value赋值为123的动作在初始化阶段执行。

　　"特殊情况"下：如果类字段的字段属性表中存在ConstantValue属性，那在准备阶段变量value就会被初始化为ConstantValue属性所指定的值，如：

```java
public static final int value = 123;
```

### 2.4 解析

　　**目的**：将常量池内的符号引用替换为直接引用。

* **符号引用**：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。符号引用与虚拟机实现的内存布局无关，引用的目标并不一定已经加载到内存中。各种虚拟机实现的内存布局可以各不相同，但是他们能接受的符号引用必须都是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。
* **直接引用**：直接引用可以是直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用是和虚拟机实现的内存布局相关的，同一个符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。如果有了直接引用，那引用的目标必定已经在内存中存在。

　　**解析时机**：为规定具体时间，只要求在执行anewarray、checkcast、getfield、getstatic、instanceof、invokedynamic、invokeinterface、invokespecial、invokestatic、invokevirtual、ldc、ldc_w、mulitanewarray、new、putfield、putstatic这16个用于操作符号引用的字节码指令之前，先对它们所使用的符号引用进行解析。（解析结果可缓存）

　　对于同一符号引用的多次解析请求，除了invokedynamic指令外，虚拟机可以对第一次解析结果进行缓存（在运行时常量池中记录直接引用，并把常量标记为已解析状态）。invokedynamic指令的目的是用于动态语言支持（仅使用Java语言不会生成这条字节码指令），它所对应的引用称为"动态调用限定符"。"动态"就是必须等到程序实际运行到这条指令是，解析动作才能进行，相对的，其余可以触发解析的指令都是"静态"的，可以在刚刚完成加载阶段，还没有开始执行代码时就进行解析。

### 2.5 初始化

　　除了加载阶段用户应用程序可以通过自定义类加载器参与之外，其余动作完全由虚拟机主导可控制。到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说是字节码）。

　　初始化阶段是执行类构造器< clinit>()方法的过程。

* < clinit>()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的。编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。

  ```java
  // 案例：非法向前引用变量
  public class Test {
      static {
          i = 0;                      // 给变量赋值可以正常编译通过
          System.out.println(i);      // 这句编译器会提示"非法向前引用"
      }
      static int i = 1;
  }
  ```

* < clinit>()方法与类的构造器函数（或者说实例构造器< init>()方法）不同，它不需要显式地调用父类构造器，虚拟机会保证在子类的< clinit>()方法执行之前，父类的< clinit>()方法已经执行完毕。因此在虚拟机中第一个被执行的< clinit>()方法的类肯定是java.lang.Object。

* 由于父类的< clinit>()方法先执行，也就以为这父类中定义的静态语句块要优先于子类的变量赋值操作。

  ```java
  // < clinit>()方法执行顺序
  public class Parent {
      public static int A = 1;
      static {A = 2;}
  }
  static class Sub extends Parent{
      public static int B = A;
  }
  public static void main(String[] args){
      System.out.println(Sub.B); // 执行结果为 2
  }
  ```

* < clinit>()方法对于类或接口来说并不是必须的，如果一个类中没有静态语句块，也没有对变量的赋值操作，那么编译器可以不为这个类生成< clinit>()方法。

* 接口中不能使用静态语句块，但仍然由变量初始化的赋值操作，因此接口与类一样都会生成< clinit>()方法。执行接口的< clinit>()方法不需要先执行父类的< clinit>()方法。只有当父接口中定义的变量使用时，父接口才会初始化。接口的实现类在初始化时一样不会执行接口的< clinit>()方法。

* 虚拟机会保证一个类的< clinit>()方法在多线程环境中被正确地解锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的< clinit>()方法，其他线程都需要阻碍等待，直到活动线程执行< clinit>()方法完毕。如果在一个类的< clinit>()方法中有耗时很长的操作，就可能造成多个进程阻塞。

***

### 3. 类加载器

　　**定义**：把类加载阶段中的"通过一个类的全限定名来获取描述此类的二进制字节流"这个动作放在Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码称为"**类加载器**"。

### 3.1 类与类加载器

　　对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都有一个独立的类名称空间。类"相等"，包括代表类的Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括使用instanceof关键字做对象所属关系判定等情况。

### 3.2 双亲委派模型

从Java虚拟机的角度看，只存在两种不同的类加载器：

* 启动类加载器（Bootstrap ClassLoader），这个类加载器使用C++语言，是虚拟机自身的一部分。
* 其他的类加载器，这些加载器是由Java语言实现，独立与虚拟机外部，并且继承于java.lang.ClassLoader。

类加载器更细致的划分：

* **启动类加载器**：这个类加载器负责将存放在< JAVA_HOME>\lib目录中的，或者被-Xbootclasspath参数所指定的路径中的，并且是虚拟机是别的（仅按照文件名识别，如rt.jar，名字不符合即使放在lib下也不被加载）类库加载到虚拟机内存中，启动类加载器无法被Java程序直接引用。用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，那直接使用null替代即可。
* **扩展类加载器**：这个加载器由sun.misc.Launcher$ExtClassLoader实现，负责加载< JAVA_HOME>\lib\ext目录中的，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可直接使用扩展类加载器。
* **应用程序类加载器**：由sun.misc.Launcher$AppClassLoader实现。由于这个类加载器是ClassLoader中的getSystemClasslLoader()方法的返回值，所以也称为系统类加载器。开发者可直接使用，如果应用程序中没有定义过自己的类加载器，一般情况下这个就是程序中**默认的类加载器**。

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/parentsdelegationmodel.png)

　　上图展示的类加载器之间的这种层次关系，称为类加载器的**双亲委派模型**。双亲委派模型要求除了顶层的启动类加载器外，其余的加载器都应该有自己的父类加载器。

　　**双亲委派模型的工作过程**：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（找不到类）时，子加载器才会自己加载。

　　**好处**：Java随着它的类加载器一起具备了一种带有优先级的层次关系。例如java.lang.Object，它存放在rt.jar之中，无论哪一个类加载器要加载这个类，最终都是委派给最顶端的启动类加载器进行加载，因此Object类在程序的各个类加载器环境中都是同一个类。　　

　　实现双亲委派的代码都集中在java.lang.ClassLoader的loadClass()方法中。代码如下：

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // 1.先检查请求的类是否已经被加载过。
    Class<?> c = findLoadedClass(name);
    if (c == null) {
        try {
            // 2.若没有加载，调用父类加载器的loadClass()方法
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                // 3.若父类加载器为空则默认使用启动类加载器做为父类加载器
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // 如果父类加载器抛出异常，说明父类加载器无法完成加载请求
        }
        if (c == null) {
            // 4.父类无法加载时，调用自身的findClass方法来进行类加载
            c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);
    }
    return c;
}
```

