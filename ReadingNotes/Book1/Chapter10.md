## 第10章 早期（编译期）优化

　　Java语言"编译期"是一个不确定的操作过程。因为它可能是指一个前端编译器把* .java文件转变成* .class文件的过程；也可能是指虚拟机后端运行期编译器（JIT编译器，Just In Time Compiler）把字节码转变成机器码的过程；还可能是指使用静态提前编译器（AOT编译器，Ahead Of Time Compiler）直接把* .java文件编译成本地机器代码的过程。

***

### 1.JavaC编译器

### 1.1 Javac的编译过程

　　**Javac的源码存放在JDK_SRC_HOME/langtools/src/share/classes/com/sun/tools/javac**中。运行com.sun.tools.javac.Main的main()方法来执行编译，和命令行中Javac的命令没什么区别。

　　从Sun Javac的代码来看，编译过程大致分为了3个过程，分别是：

* 解析与填充符号表过程。
* 插入式注解处理器的注解处理过程。
* 分析与字节码生成过程。

　　这个三个步骤的关系与交互图顺序（**图：Javac编译过程**）如下图所示：

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/javaccompilerprocess.png)

　　Javac编译动作的入口是com.sun.tools.javac.main.JavaCompiler类。上述3个过程代码逻辑集中在这个类的compile()和compile2()方法中。**Javac编译过程的主体代码**如下图：

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/javaccompilercode.png)

### 1.2 解析与填充符号表

　　解析步骤由上图的parseFiles()方法（过程1.1）完成。解析步骤包括词法分析和语法分析两个过程。

* **词法分析**是将源代码的字符流转变为标记（Token）集合。单个字符是程序编写过程的最小元素，而标记这是编译过程的最小元素，关键字、变量名、字面量、运算符都可以称为标记，如"int a=b+2"这句代码包含了6个标记，分别是int、a、=、b、+、2，虽然关键字int由3个字符构成，但是它是一个Token，不可再拆分。Java源码中，此法分析过程由com.sun.tools.javac.parser.Scanner类来实现。
* **语法分析**是根据Token序列构造抽象语法树的过程。抽象语法树（AST）是一种用来描述程序代码语法结构的树形表示方式，语法树的每一个节点都代表这程序代码中的一个语法结构，例如包、类型、修饰符、运算符、接口、返回值甚至代码注释等都可以是一个语法结构。语法分析过程由com.sun.tools.javac.parser.Parser类来实现，这个阶段产生的抽象语法树由com.sun.tools.javac.tree.JCTree类来表示。

　　解析符号表由上图的enterTree()方法（过程1.2）来执行。符号表是由一组符号地址和符号信息构成的表格，在语义分析中，符号表所登记的内容将用于语义检查（如检查一个名字的使用和原先的说明是否一致）和产生中间代码。在目标代码生成阶段，当对符号名进行地址分配时，符号表时地址分配的依据。填充符号表的过程由com.sun.tools.javac.comp.Enter类实现。

### 1.3 注解处理器

　　插入式**注解处理器**在编译期间对注解进行处理，可以看作插件，在插件中，可以读取、修改、添加抽象语法树的人艺元素。如果这些插件在处理注解期间对语法树进行了修改，编译器将回到解析及填充符号表的过程重新处理，知道所有插入式注解处理器都没有再对语法树进行修改位置，每一次循环称为一个Round，也就是（图：Javac编译过程）中的回环过程。

　　在Java源码中，插入式注解处理器：

* 初始化过程是在initPorcessAnnotations()方法中完成
* 执行过程是在processAnnotations()方法中完成，这个方法判断是否还有新的注解处理器需要执行
* 如果有，通过com.sun.tools.javac.processing.JavaProcessingEnvironment类的doProcessing()方法生成一个新的JavaCompiler对象对编译的后续步骤进行处理。

### 1.4 语义分析和字节码(Class文件)生成

　　**语义分析**的主要任务是对结构上正确的源程序进行上下文有关性质的审查，如进行类型审查。语义分析分为标注检查和数据及控制流分析两个步骤，分别由上图的attribute()和flow()方法（过程3.1和过程3.2）完成。

* **标注检查**步骤检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否匹配等。

  ```java
  int a = 1 + 2;
  ```

  如上述代码，语法树上仍然能看到字面量"1","2"以及操作符"+"，但是经过**常量折叠**后，它们将会被折叠为字面量"3"。由于编译器见进行了常量折叠，所以代码里面定义"a=1+2"比起直接定义"a=3"，并不会增加程序运行期间CPU指令的运算量。

  标注检查步骤源码的实现类是com.sun.tools.javac.comp.Attr类和com.sun.tools.javac.comp.Check类。

* **数据及控制流分析**是对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理了等问题。

  **eg**：关于final修饰符的数据流及控制流分析的例子

  ```java
  public void foo(final int arg) {
      final int var = 0;
      // do something
  }
  public void foo(int arg) {
      int var = 0;
      // do something
  }
  ```

  两段代码编译出来的Class文件无区别，局部变量与字段（实例变量、类变量）是有区别的，它在常量池中没有CONSTANT_Fieldref_info的符号引用，自然就没有访问标志（Access_Flags）的信息，甚至可能连名称都不会保留，自然在Class文件中不可能知道一个局部变量是不是声明为final了。

  将局部变量声明为final，对运行期没有影响，变量的不变性仅仅由编译器在编译期间保障。数据及控制流分析的入口是上图的flow()方法（过程3.2），由com.sun.tools.javac.comp.Flow类完成。

* Java中最常见的语法糖由泛型、变长参数、自动装箱/拆箱等，虚拟机运行时不支持这些语法，在编译阶段还原回简单的基础语法结构，这个过程叫**解语法糖**。解语法糖过程由desugar()方法触发，在com.sun.tools.javac.comp.TransTypes类和com.sun.tools.javac.comp.Lower类中完成。

* **字节码生成**是编译过程最后一个阶段，由com.sun.tools.javac.jvm.Gen类完成。字节码生成阶段不仅仅是把前面各个步骤生成的信息（语法树、符号表）转化成字节码写到磁盘中，编译器还进行了少量的代码添加和转换工作。例如实例构造器< init>()方法和类构造器< clinit>()方法就是这个阶段添加到语法树之中的。例如把字符串的加操作替换为StringBuilder的append()操作等。

  完成对语法树的遍历和调整后，会把填充了所有所需信息的符号表交给com.sun.tools.javac.jvm.ClassWriter类，由这个类的writeClass()方法输出字节码，生成最终的Class文件。到此为止编译过程结束。

***

### 2 Java语法糖的味道

### 2.1 泛型和类型擦除

　　泛型是JDK1.5新增特性，它的本质是参数化类型的应用，也就是会所操作的数据类型被指定为一个参数。

　　泛型技术在C#和Java之中的使用比较：

* C#里泛型无论在程序源码中、编译后的IL中，或是运行期的CLR中，都是切实存在的，List< int>与List< String>就是两个不同的类型，它们在系统运行期生成，有自己的虚方法表和数据类型，这种实现称为**类型膨胀**，基于这种方法实现的泛型称为**真实泛型**。
* Java中的泛型，只在源码中存在，编译后的字节码文件中，就已经替换为原来的原生类型（裸类型）了，并且相应的地方插入了强制转型代码，因此，对于运行期的Java而言，ArrayList< int>与ArrayList< String>就是同一个类。Java语言中泛型的实现方法称为**类型擦除**，基于这种方法实现的泛型称为**伪类型**。

**eg**：当泛型遇见重载　　

```java
public class GenericTypes {
    public static void method(List<String> list){
        System.out.println("invoke method(List<String> list)");
    }
    public static void method(List<Integer> list){
        System.out.println("invoke method(List<Integer> list)");
    }
}
```

　　上面的代码无法编译，因为编译后变成了一样的原生类型List< E>，擦除动作导致这两个方法的特征签名变得一模一样。泛型擦出成相同的原生类型，只是无法重载的其中一部分原因，请看下面代码：

```java
// JDK1.6环境可以编译通过，其他编译器可能会拒绝编译
public class GenericTypes {
    public static String method(List<String> list){
        System.out.println("invoke method(List<String> list)");
        return "";
    }
    public static int method(List<Integer> list){
        System.out.println("invoke method(List<Integer> list)");
        return 1;
    }
}
```

　　方法重载要求方法具备不同的特征签名，返回值并不包含在方法的特征签名之中，所以返回值不参与重载，但是在Class文件之中，只要描述符不是完全一致的两个方法就可以共存。

　　Signture的作用是存储一个方法在字节码层面的特征签名，这个属性保存的参数类型并不是原生类型，而是包括了参数化类型的信息。特征签名最重要的任务就是做为方法独一无二且不可重复的ID，在Java代码中的方法特征签名只包括了方法名称、参数顺序以及参数类型，而在字节码中的特征签名还包括方法返回值及受查异常表。

　　擦除法所谓的擦除，仅仅是对方的Code属性中的字节码进行擦除，实际上元数据还是保留了泛型信息，这也是我们能通过反射手段取得参数化类型的根本依据。

### 2.2 自动装箱、拆箱与遍历循环

**eg**：自动装箱、拆箱与遍历循环

```java
public static void main(String[] args){
    List<Integer> list = Arrays.asList(1, 2, 3, 4);
    int sum = 0;
    for (int i : list) {
        sum += i;    
    }
    System.out.println(sum);
}
```

**eg**：自动装箱、拆箱与遍历循环编译之后

```java
public static void main(String[] args){
    List<Integer> list = Arrays.asList(new Integer[] {
            Integer.valueOf(1), Integer.valueOf(2),
            Integer.valueOf(3), Integer.valueOf(4)});
    int sum = 0;
    for (Iterator localIterator = list.iterator; localIterator.hasNext();) {
        int i = ((Integer)localIterator.next()).intValue();
        sum += i;
    }
    System.out.println(sum);
}
```

编译之后，遍历循环把代码还原成了迭代器的实现，这也是为何遍历循环需要被遍历的类实现Iterable接口。

**eg**：自动装箱的陷阱

```java
public static void main(String[] args) {     
    Integer a = 1;
    Integer b = 2;
    Integer c = 3;
    Integer d = 3;
    Integer e = 321;
    Integer f = 321;
    Long g = 3L;
    System.out.println(c == d);          // true
    System.out.println(e == f);          // false
    System.out.println(c == (a + b));    // true
    System.out.println(c.equals(a + b)); // true
    System.out.println(g == (a + b));    // true
    System.out.println(g.equals(a + b)); // false
}
```

### 2.3 条件编译

　　Java语言也可以进行条件编译，方法就是使用条件为常量的if语句。如下代码，此代码中的if语句不同于其他java代码，它在编译阶段就会被运行，生成字节码之中只包括"System.out.println("block 1")"一条语句。

```java
public static void main(String[] args) {
    if(true){
        System.out.println("block 1");
    } else {
        System.out.println("block 2");
    }
}
```

上述代码编译后的Class文件的反编译结果：

```java
public static void main(String[] args) {
    System.out.println("block 1");
}
```

***

### 3.插入式注解处理器

　　实现注解处理器需要继承抽象类javax.annotaton.processing.AbstractProcessor，这个抽象类中只有一个必须覆盖的abstract方法："process()"，它是Javac编译器在执行注解处理器代码时要调用的过程，我们可以从这个方法的第一个参数"annotations"中获取到此注解处理器所要处理的注解集合，从第二个参数"roundEnv"中访问到当前这个Round中的语法树节点，每个语法树节点在这里表示为一个Element。除了process()方法传入参数之外，还有一个很常用的实例变量"processingEnv"，它是AbstractProcessor中的一个protected变量，在注解处理器初始化的时候（init()方法执行的时候）创建，继承了AbstractProcessor的注解处理器代码可以直接访问它。它代表了注解处理器框架提供的一个上下文环境，要创建新的代码、向编译器输出信息、获取其他工具类等都需要用到这个实例变量。

　　注解处理器除了process()方法及其参数外，还有两个可以配合使用的Annotations：@SupportedAnnotationTypes和@SupportedSourceVersion，前者代表了这个注解处理器对哪些注解感兴趣，可以使用星号"*"做为通配符代表对所有的注解都感兴趣，后者指出这个注解处理器可以处理哪些版本的Java代码。

　　每一个注解处理器在运行的时候都是单例的，如果不需要改变或生成语法树的内容，process()方法就可以返回一个值为false的布尔值，通知编译器这个Round中的代码未发生变化，无需构造新的JavaCompiler实例。