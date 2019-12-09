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
  public class NotInitialization {
  	public static void main(String[] args) {
  		SuperClass[] sca = new SuperClass[10];
  	}	
  }
  ```

  

　　

　　

