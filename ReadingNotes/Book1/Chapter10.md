## 第10章 早期（编译期）优化

　　Java语言"编译期"是一个不确定的操作过程。因为它可能是指一个前端编译器把* .java文件转变成* .class文件的过程；也可能是指虚拟机后端运行期编译器（JIT编译器，Just In Time Compiler）把字节码转变成机器码的过程；还可能是指使用静态提前编译器（AOT编译器，Ahead Of Time Compiler）直接把* .java文件编译成本地机器代码的过程。

***

### 1.JavaC编译器

### 1.1 Javac的源码与调试

　　**Javac的源码存放在JDK_SRC_HOME/langtools/src/share/classes/com/sun/tools/javac**中。运行com.sun.tools.javac.Main的main()方法来执行编译，和命令行中Javac的命令没什么区别。

　　从Sun Javac的代码来看，编译过程大致分为了3个过程，分别是：

* 解析与填充符号表过程。
* 插入式注解处理器的注解处理过程。
* 分析与字节码生成过程。

　　这个三个步骤的关系与交互图顺序如下图所示：

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/javaccompilerprocess.png)

　　Javac编译动作的入口是com.sun.tools.javac.main.JavaCompiler类。上述3个过程代码逻辑集中在这个类的compile()和compile2()方法中。主体代码如下图：

![](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/javaccompilercode.png)

### 1.2 解析与填充符号表



