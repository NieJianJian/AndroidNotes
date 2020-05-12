## 内部类

### 目录

* [1. 简介](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/InnerClass.md#1-简介)
* [2. 普通内部类](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/InnerClass.md#2-普通内部类)
* [3. 静态内部类](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/InnerClass.md#3-静态内部类)
* [4. 匿名内部类](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/InnerClass.md#4-匿名内部类)
* [5. 局部内部类](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/InnerClass.md#5-局部内部类)
* [6. 深入了解内部类](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/InnerClass.md#6-深入了解内部类)

***

### 1. 简介

内部类，无非就是在类的内部定义一个类。

* 为什么要使用内部类？
  * 独立结构：每个内部类都能独立地继承一个（接口的）实现，所以无论外围类是否已经继承了某个（接口的）实现，对于内部类都没有影响。
  * 多继承：接口只是解决了部分问题，而内部类使得多重继承的解决方案变得更加完整。

* 内部类特性（摘自《Think in java》）

  * 内部类可以用多个实例，每个实例都有自己的状态信息，并且与其他外围对象的信息相互独立。

  * 在单个外围类中，可以让多个内部类以不同的方式实现同一个接口，或者继承同一个类。

  * 创建内部类对象的时刻并不依赖于外围类对象的创建。
  * 内部类并没有令人迷惑的“is-a”关系，他就是一个独立的实体。
  * ***内部类是个编译时的概念，在编译期会被编译为跟外部类一样的顶级类***。对于一个名为OuterClass的外围类和一个名为InnerClass的内部类，在编译成功后，会出现这样两个class文件：OuterClass.class和OuterClass$InnerClass.class。

* 内部类的分类
  * 普通内部类
  * 静态内部类
  * 匿名内部类
  * 局部内部类
* 内部类的嵌套
  - 普通内部类：在这里我们可以把它看成一个外部类的普通成员方法，在其内部可以定义普通内部类（嵌套的普通内部类），但是无法定义 static 修饰的内部类，就像你无法在成员方法中定义 static 类型的变量一样，当然也可以定义匿名内部类和局部内部类；
  - 静态内部类：因为这个类独立于外部类对象而存在，我们完全可以将其拿出来，去掉修饰它的 static 关键字，他就是一个完整的类，因此在静态内部类内部可以定义普通内部类，也可以定义静态内部类，同时也可以定义 static 成员；
  - 匿名内部类：和普通内部类一样，定义的普通内部类只能在这个匿名内部类中使用，定义的局部内部类只能在对应定义域内使用；
  - 局部内部类：和匿名内部类一样，但是嵌套定义的内部类只能在对应定义域内使用。

***

### 2. 普通内部类

```java
public class OuterClass1 {
    public class InnerClass {
    }
}
```

OuterClass1为外部类，InnerClass为内部类。

* OuterClass1类之内，直接使用`new InnerClass()`创建内部类对象。

* OuterClass1类之外，首先先创建外部类对象，然后使用`外部类对象.new 内部类构造器()`的方式创建。

  ```java
  OuterClass1 outClass = new OuterClass1();
  OuterClass1.InnerClass innerClass = outClass.new InnerClass();
  ```

下面看一个简单案例。

```java
public class OuterClass1 {
    int field1 = 1;
    private int field2 = 2;
    public int field3 = 3;
    public OuterClass1() {
        InnerClass innerClass = new InnerClass();
        System.out.println("内部类field5的值： " + innerClass.field5);
        System.out.println("内部类field6的值： " + innerClass.field6);
        System.out.println("内部类field7的值： " + innerClass.field7);
    }
    public class InnerClass {
        int field5 = 5;
        private int field6 = 6;
        public int field7 = 7;
        // static int field8 = 8; // 编译错误！普通内部类中不能定义 static 属性
        public InnerClass() {
            System.out.println("外部类field1的值： " + field1);
            System.out.println("外部类field2的值： " + field2);
            System.out.println("外部类field3的值： " + field3);
        }
    }
    public static void main(String[] args) {
        OuterClass1 outerClass1 = new OuterClass1();
    }
}
```

运行结果：

```
外部类field1的值： 1
外部类field2的值： 2
外部类field3的值： 3
内部类field5的值： 5
内部类field6的值： 6
内部类field7的值： 7
```

上述代码和运行结果可以看出：

* 内部类中可以访问外部类对象中的所有访问权限的字段。
* 在外部类中，可以通过内部类的对象访问内部类中所有访问权限的字段。
* 普通内部类不能存在任何static的变量和方法。
* 如果在OuterClass1类外部调用InnerClass，访问权限和普通类与类之间的访问权限一致。

***

### 3. 静态内部类

静态内部类和非静态内部类（普通内部类）区别：

* 非静态内部类编译完成会隐式的持有外部类的引用，静态内部类则没有。
* 非静态内部类创建需要先创建外部类，静态内部类不需要。
* 静态内部类不能访问任何外部类中非static的成员变量和方法。

下面看一个简单的案例。

```java
public class OuterClass2 {
    private int field1 = 1;
    public OuterClass2() {
        OuterClass2.InnerClass innerClass = new OuterClass2.InnerClass();
        System.out.println("内部类field2的值： " + innerClass.field2);
        System.out.println("内部类field3的值： " + InnerClass.field3);
    }
    static class InnerClass {
        int field2 = 2;
        static int field3 = 3;
        public InnerClass() {
            // System.out.println("外部类field3的值： " + field1); // 编译错误
        }
    }
    public static void main(String[] args) {
        OuterClass2 outerClass2 = new OuterClass2();
        // 无需先创建外部类对象，直接创建内部类对象就可以
        // OuterClass2.InnerClass innerClass = new OuterClass2.InnerClass();
    }
}
```

运行结果：

```
内部类field2的值： 2
内部类field3的值： 3
```

***

### 4. 匿名内部类

匿名内部类就是没有名字的类。其中最常见的一种形式莫过于在方法参数中新建一个接口对象 / 类对象，并且实现这个接口声明 / 类中原有的方法了：

```java
public class OuterClass4 {
    private int field1 = 1;
    public OuterClass4() {
    }
    // 自定义接口
    interface OnClickListener {
        void onClick(Object obj);
    }
    private void anonymousClassTest() {
        final int field2 = 2; // 被匿名内部类使用，需要final修饰
        int field3 = 3; // 没有被匿名内部类使用，不需要final修饰。
        // 在这个过程中会新建一个匿名内部类对象，
        // 这个匿名内部类实现了 OnClickListener 接口并重写 onClick 方法
        OnClickListener clickListener = new OnClickListener() {
            // 可以在内部类中定义属性，但是只能在当前内部类中使用，
            // 无法在外部类中使用，因为外部类无法获取当前匿名内部类的类名，
            // 也就无法创建匿名内部类的对象
            int field = 1; // 外部类无法调用该变量
            @Override
            public void onClick(Object obj) {
                System.out.println("其外部类的 field1 字段的值为: " + field1);
                System.out.println("局部变量 field2 字段的值为: " + field2);
            }
        };
        // new Object() 过程会新建一个匿名内部类，继承于 Object 类，并重写了 toString() 方法
        clickListener.onClick(new Object() {
            @Override
            public String toString() {
                return "obj1";
            }
        });
    }
    public static void main(String[] args) {
        OuterClass4 outerClass4 = new OuterClass4();
        outerClass4.anonymousClassTest();
    }
}
```

上面的代码中展示了常见的两种使用匿名内部类的情况： 

* 直接 new 一个接口，并实现这个接口声明的方法，在这个过程其实会创建一个匿名内部类实现这个接口，并重写接口声明的方法，然后再创建一个这个匿名内部类的对象并赋值给前面的OnClickListener类型的引用。 

* new一个已经存在的类 / 抽象类，并且选择性的实现这个类中的一个或者多个非 final 的方法，这个过程会创建一个匿名内部类对象继承对应的类 / 抽象类，并且重写对应的方法。

我们需要注意几个地方：

* 匿名内部类没有访问修饰符。匿名内部类不能定义构造函数。
* 在匿名内部类中可以使用外部类的属性，但是外部类却不能使用匿名内部类中定义的属性，因为是匿名内部类，因此在外部类中无法获取这个类的类名，也就无法得到属性信息。
* 被匿名内部类使用到的局部变量（如field2），需要final修饰，否则会报错。因为编译期传入到匿名内部类中的示拷贝，如果不是final，外部对变量进行了修改，显然示有问题的。JDK 8不需要手动加上final，系统会自行判断是否需要添加。
* 被匿名内部类使用到的外部类成员变量（如field1），不需要final修饰。因为匿名内部类中使用外部类成员变量是通过this进行访问的。

**2. 匿名内部类的初始化**

匿名内部类不能定义构造函数，但是可以使用代码块进行初始化。

```java
public InnerClass getInnerClass() {
    return new InnerClass() {
        int age;
        // 使用代码块来完成初始化工作
        {
            age = 20;
        }
    };
}
```

***

### 5. 局部内部类

局部内部类用的比较少，一般声明在一个方法体 / 一段代码块中。

* 同样的，局部内部类中可以访问外部类对象中的所有访问权限的字段。
* 外部类不能访问局部内部类中定义的字段，因为局部内部类的定义只在其特定的方法体 / 代码块中有效，一旦出了这个定义域，那么其定义就失效了。

下面看一个简单的案例。

```java
public class OuterClass3 {
    public OuterClass3() {
    }
    private void localInnerClassTest() {
        // 局部内部类 A，只能在当前方法中使用
        class A {
            // static int field = 1; // 编译错误！局部内部类中不能定义 static 字段
            public A() {
            }
        }
        A a = new A();
        if (true) {
            // 局部内部类 B，只能在当前代码块中使用
            class B {
                public B() {
                }
            }
            B b = new B();
        }
        // B b1 = new B(); // 编译错误！不在类 B 的定义域内，找不到类 B，
    }
    public static void main(String[] args) {
        OuterClass3 outerClass3 = new OuterClass3();
        outerClass3.localInnerClassTest();
    }
}
```

***

### 6. 深入了解内部类

#### 6.1 内部类和外部类互相访问

　　内部类和外部类一样是顶级类，那不是意味着对方是有的method、field示无法被访问得到的。事实上外部类为了访问内部类私有的域/方法，编译期间自动会为内部类生成`access$`数字编号相关方法。

* 避免编译期自动生成`access$**`相关方法：
  * 一个外部类如果有内部类，把所有method/field的私有访问权限改成protected或public或者默认访问权限。
  * 同时把内部类的所有method/field的私有访问权限改成protected或public或者默认访问权限。

下面看一个简单案例。

```java
public class OuterClass {
    int field1 = 1;
    private int field2 = 2;
    public int field3 = 3;
    public void getInnerClass() {
        InnerClass innerClass = new InnerClass();
        System.out.println("变量 field4 的值为 ： " + innerClass.field4);
        System.out.println("变量 field5 的值为 ： " + innerClass.field5);
        System.out.println("变量 field6 的值为 ： " + innerClass.field6);
    }
    class InnerClass {
        int field4 = 4;
        private int field5 = 5;
        public int field6 = 6;
        public InnerClass() {
            System.out.println("变量 field1 的值为 ： " + field1);
            System.out.println("变量 field2 的值为 ： " + field2);
            System.out.println("变量 field3 的值为 ： " + field3);
        }
    }
    public static void main(String[] args) {
        new OuterClass().getInnerClass();
    }
}
```

外部类 / 内部类，都分别定义了三种访问权限的变量，并且在内部类 / 外部类进行了调用。

* `javac OuterClass.java` 进行编译，会生成两个class文件，分别是`OuterClass$InnerClass`和`OuterClass`，这就印证了**内部类会生成和外部一样的顶级类**。

接下来看编译后的class文件，

```java
public class OuterClass {
    int field1 = 1;
    private int field2 = 2;
    public int field3 = 3;
    public OuterClass() {
    }
    public void getInnerClass() {
        InnerClass var1 = new InnerClass(this);
        System.out.println("变量 field4 的值为 ： " + var1.field4);
        System.out.println("变量 field5 的值为 ： " + InnerClass.access$000(var1));
        System.out.println("变量 field6 的值为 ： " + var1.field6);
    }
    public static void main(String[] var0) {
        (new OuterClass()).getInnerClass();
    }
}
```

```java
class OuterClass$InnerClass {
    int field4;
    private int field5;
    public int field6;
    public OuterClass$InnerClass(OuterClass var1) {
        this.this$0 = var1;
        this.field4 = 4;
        this.field5 = 5;
        this.field6 = 6;
        System.out.println("变量 field1 的值为 ： " + var1.field1);
        System.out.println("变量 field2 的值为 ： " + OuterClass.access$100(var1));
        System.out.println("变量 field3 的值为 ： " + var1.field3);
    }
}
```

* 首先看`OuterClass$InnerClass`，构造方法中将OuterClass作为参数传入，并赋值给了`this$0`，这也就印证了**普通内部类会持有外部类的引用**。
* 接着看变量调用的部分，会看到调用**private**权限修饰的变量时，调用的是名为`access$数字编号`静态方法。这也就印证了**内部类 / 外部类调用外部类 / 内部类的私有的域或者方法时，会在外部类 / 内部类生成名为`access$数字编号的`的静态方法**。

字节码层面验证（只显示重要信息，省略不重要的行数）

```
javap -c OuterClass\$InnerClass.class 
Compiled from "OuterClass.java"
class com.nj.test.OuterClass$InnerClass {
  final com.nj.test.OuterClass this$0;
  public com.nj.test.OuterClass$InnerClass(com.nj.test.OuterClass);
    Code:
       2: putfield      #2                  // Field this$0:Lcom/nj/test/OuterClass;
       
      69: invokestatic  #16                 // Method com/nj/test/OuterClass.access$100:(Lcom/nj/test/OuterClass;)I
     109: return
     
  static int access$000(com.nj.test.OuterClass$InnerClass);
    Code:
       0: aload_0
       1: getfield      #1                  // Field field5:I
       4: ireturn
}
```

* 首先我们看到，内部类中定义了一个OuterClass 类型的变量`this$0`
* 构造函数接收一个OuterClas类型的参数，也就是外部类的对象。
* 构造函数初始化过程中，第2行调用putfield指令为`this$0`进行赋值。
* 第69行，invokestatic指令说明调用的是静态方法，方法名为`access$100`，并且接收一个OuterClass类型的参数，返回值是一个int类型。
* 最后，InnerClass内部生成了一个静态方法`access$000`，可以看一下上面的OuterClass文件中，其实就是为外部类提供的调用私有变量的方法。对应的外部类也会生成静态方法`access$100`供内部类访问外部类的私有变量使用。

**总结**：在非静态内部类访问外部类私有成员 / 外部类访问内部类私有成员 的时候，对应的外部类 / 外部类会生成一个静态方法，用来返回对应私有成员的值，而对应外部类对象 / 内部类对象通过调用其内部类 / 外部类提供的静态方法来获取对应的私有成员的值。

#### 6.2 匿名内部类的编译

　　匿名内部类的名称格式一般是`外部类$数字编号`，后面的数字编号，就是根据该匿名内部类的外部类中出现的先后关系，依次累加命名的。热修复补丁新增匿名内部类会导致方法的新增。

　　如果有补丁热部署的需求，应该极力避免插入一个新的匿名内部类。当然如果匿名内部类示插入到外部类的末尾，那么是允许的。

***

### 7. 内部类与内存泄露

Android建议Handler的实现尽量使用静态内部类，防止外部Activity类不能被回收导致可能的内存泄露。

***

### 参考链接

* [回归Java基础，详解Java内部类](https://mp.weixin.qq.com/s/wMa4q0AOFS8b57c-Qb_kuQ)
* [详解内部类](https://www.cnblogs.com/chenssy/p/3388487.html)
* [实现多重继承](https://www.cnblogs.com/chenssy/p/3389027.html)
* [详解匿名内部类](https://www.cnblogs.com/chenssy/p/3390871.html)
* [匿名内部类的这几个骚操作，99% 的人都不知道](https://mp.weixin.qq.com/s/qIpwEjRd2rYEze6KzC6o6Q)
* [匿名内部类访问的外部类局部变量为什么要用final 修饰，jdk8为啥不需要了](https://www.wanandroid.com/wenda/show/8836)

