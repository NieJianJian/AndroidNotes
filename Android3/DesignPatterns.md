## 设计模式

### 目录

* [面向对象六大原则](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#面向对象六大原则)
* [设计模式分类](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#设计模式分类)
* [单例模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#单例模式)
* [Builder模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#builder模式)
* [工厂模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#工厂模式)
* [抽象工厂模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#抽象工厂模式)
* [策略模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#策略模式)
* [责任链模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#责任链模式)
* [观察者模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#观察者模式)
* [代理模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#代理模式)
* [装饰模式](https://github.com/NieJianJian/AndroidNotes/blob/master/Android3/DesignPatterns.md#装饰模式)

***

***

***

### 面向对象六大原则

* 单一职责原则：SRP。

  一个应该仅有一个引起它变化的原因。简单来说，一个类应该是一组相关性很高的函数、数据的封装。

* 开闭原则：OCP。

  一个类、模块、函数等应该对于扩展开放，对于修改是封闭的。遵循开闭原则的重要手段是抽象。

* 里氏替换原则：LSP。

  使用父类的地方都能替换成子类。

* 依赖倒置原则：DIP。

  是指一种特定的解耦形式。模块间的依赖通过抽象产生，实现类之间不发生直接的依赖关系，其依赖关系是通过接口或抽象类产生的。

  * 高层模块不应该依赖于低层模块，两者都应该依赖其抽象；
  * 抽象不应该依赖细节，细节应该依赖于抽象。

* 接口隔离原则：ISP。

  不应该依赖于它们不需要的接口。

* 最少知识原则：LOD。

  一个对象应该对其他对象有最少的了解。

***

***

***

### 设计模式分类

* 创建型模式

  　　创建型模式，就是创建对象的模式，抽象了实例化的过程。 它帮助一个系统独立于如何创建、组合和表示它的那些对象。 关注的是对象的创建，创建型模式将创建对象的过程进行了抽象，也可以理解为将创建对象的过程进行了封装，作为客户程序仅仅需要去使用对象，而不再关心创建对象过程中的逻辑。

  代表：**工厂方法模式、抽象工厂模式、单例模式、建造者模式、原型模式**。

* 行为型模式

  　　行为型模式是对在不同的对象之间划分责任和算法的抽象化，行为型模式不仅仅关注类和对象的结构，而且重点关注他们之间的相互作用，通过行为型模式，可以更加清晰地划分类与对象的职责，并研究系统在运行时实例对象之间的交互。

  代表：**策略模式、模板方法模式、观察者模式、迭代子模式、责任链模式、命令模式、备忘录模式、状态模式、访问者模式、中介者模式、解释器模式**。

* 结构型模式

  　　结构型模式是为解决怎样组装现有的类，设计他们的交互方式，从而达到实现一定的功能。

  代表：**适配器模式、装饰器模式、代理模式、外观模式、桥接模式、组合模式、享元模式**。

**以上23种设计模式，都是建立在面向对象的基础上的**。

***

***

***

### 单例模式

#### 1. 定义

确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个类。

#### 2. 使用场景

* 确保某个类只有一个对象的场景，避免产生多个对象消耗过多资源。
* 某种类型的对象应该有且只有一个。如要访问IO和数据库等资源。

#### 3. 实现的方式

实现单例模式主要的关键点：

* 构造函数不对外开放，一般为Private；（私有化，不能通过new来手动构造）
* 通过一个静态方法或者枚举返回单例类对象；
* 确保单例类的对象有且只有一个，尤其是在多线程环境下；（确保线程安全）
* 确保单例类对象在序列化时不会重新构建对象。

##### 3.1 懒汉模式

```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

* 优点：使用的时候才被实例化，节省资源。
* 缺点：第一次加载反应稍慢，每次都调用同步，造成不必要的开销。

##### 3.2 饿汉式

```java
public class Singleton {
    private static Singleton instance = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() {
        return instance;
    }
}
```

* 优点：不用加锁，线程安全，执行效率高。
* 缺点：类加载时就初始化，浪费资源。

##### 3.3 双重检验锁（DCL）

```java
public class Singleton {
    private volatile static Singleton instance = null;
    private Singleton() {}
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

* 优点：需要的时候才初始化，节省资源；还能保证线程安全；且实例化后不再需要同步锁。

`instance = new Singleton()`不是原子操作，会汇编指令分析，它会做以下3件事情：

```java
inst = allocat(); // 为Singleton的实例分配内存
constructor(inst); // 调用Singleton的构造函数，初始化成员字段
instance = inst; // 将instance对象指向分配的内存空间（此时instance就不是null了）
```

　　由于Java编译器允许处理器乱序执行，所以第二步和第三步的顺序无法保证。如果第三步先执行完毕、第二步未执行时，有另外的线程调用了instance，由于已经赋值，将判断不为null，拿去直接使用，但其实构造函数还未执行，成员变量等字段都未初始化，直接使用，就会报错。这就是**DCL失效问题**，而且很难复现。
**volatile**有禁止指令重排序的功能。（具体参考[Java并发三大特性](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/JavaConcurrent.md)）

##### 3.4 静态内部类单例

```java
public class Singleton {
    private Singleton() {}
    public static Singleton getInstance() {
        return SingletonInstance.instance;
    }
    private static class SingletonInstance {
        private static final Singleton instance = new Singleton();
    }
}
```

1. 外部类加载时不会去加载内部类，内部类不被加载，就不会初始化instance，所以不会占内存，延迟初始化。
2. 类的静态变量会在类加载（类初始化构造器< clinit>调用）的时候初始化，Java虚拟机保证< clinit>方法只会调用一次，并且在多线程环境下可以被正确的加锁、同步，所以可以保证线程安全。

##### 3.5 枚举单例

```java
public enum Singleton {
    INSTANCE;
    public void doSomething(){
        ...
    }
}
```

优点：实现简单；线程安全；任何情况下都是一个单例。

前面的几种方法有可能出现重新创建对象的可能，那就是序列化。

序列化将单例的实例对象写到磁盘，然后在都会来，从而有效的获得一个实例。即使函数的构造方法是私有的，反序列化时依然可以通过特殊的途径去创建类的一个新的实例，相当于调用了该类的构造函数。反序列化操作提供了一个很特别的钩子函数，类中具有一个私有的、被实例化的方法readResolve()，这个方法可以让开发人员控制对象的反序列化。为了保证反序列化的过程中仍然保持单例的特性，可以在单例中添加一个readResolve()方法

```java
private Object readResolve() throws ObjectStreamException {
    return instance;
}
```

在反序列化从I/O流中读取读取对象时，readResolve()方法会被调用，实际上就是用readResolve()中返回的对象直接替换掉在反序列化中创建的对象。

##### 3.6 容器单例

```java
public class Singleton {
    private static Map<String, Object> instanceMap = new HashMap<String, Object>();
    private Singleton() {}
    public static void addInstance(String key, Object instance) {
        if (!instanceMap.containsKey(key)) {
            instanceMap.put(key, instance);
        }
    }
    public static Object getInstance(String key) {
        return instanceMap.get(key);
    }
}
```

在程序初始化的时候，将多种单例类型注入到一个统一的管理类中，根据key获取对应的对象。

#### 4. 源码中的应用

* LayoutInflater类

  Android中经常通过你过Context获取系统级别的服务，这些服务会在合适的时候以单例的形式注册到系统中，需要的时候通过Context的getSystemService(String name)获取。

***

***

***

### Builder模式

#### 1. 定义

将一个复杂对象的构建与它的表示分离，是的同样的构建过程可以创建不同的表示。

#### 2. 使用场景

* 相同的方法，不同的执行顺序，产生不同的事件结果。
* 多个部件或零件，都可以装配到一个对象中，但是产生的运行结果又不相同时。
* 产品类非常复杂，或者产品类中的调用顺序不同产生了不同的作用，这个时候使用建造者模式非常合适。
* 当初始化一个对象特别复杂，如参数多，且很多参数都具有默认值时。

#### 3. Android源码中的实现

Android源码中，最常用到的Builder模式就是AlertDialog.Builder，使用该Builder用来组装Dialog的各个部分，从而构建复杂的AlertDialog。

```java
//显示基本的AlertDialog  
private void showDialog(Context context) {  
    AlertDialog.Builder builder = new AlertDialog.Builder(context);  
    builder.setIcon(R.drawable.icon);  
    builder.setTitle("Title");  
    builder.setMessage("Message");  
    builder.setPositiveButton("Button1",  
            new DialogInterface.OnClickListener() {  
                public void onClick(DialogInterface dialog, int whichButton) {  
                    setTitle("点击了对话框上的Button1");  
                }  
            });  
    builder.setNeutralButton("Button2",  
            new DialogInterface.OnClickListener() {  
                public void onClick(DialogInterface dialog, int whichButton) {  
                    setTitle("点击了对话框上的Button2");  
                }  
            });  
    builder.setNegativeButton("Button3",  
            new DialogInterface.OnClickListener() {  
                public void onClick(DialogInterface dialog, int whichButton) {  
                    setTitle("点击了对话框上的Button3");  
                }  
            });  
    builder.create().show();  // 构建AlertDialog， 并且显示
} 
```

#### 4. 源码中的应用

* AlertDialog.Builder

#### 5. 总结

Java设置参数的方式：

* **重叠构造器模式**：一个类有多个构造器，构造器的参数个数递增，层层调用。

  缺点：参数比较多时，会涉及到参数顺序的问题，很有可能传参出错，难以阅读，也很难把控；并且很多不想设置的参数，不得不为它们传值。<sup>[1]</sup>

* **JavaBean模式**：调用一个无参构造器来创建对象，然后调用setter方法来设置参数。

  优点：弥补了重叠构造器的不足，创建实例很容易，代码也容易阅读。

  缺点：JavaBean可以随时调用setter方法进行设置，导致JavaBean缺乏一致性，也就是类的实例的一致性无法保证。特别是多线程情况下，会出现很难分析的bug。

* **Builder模式**：不直接生成想要的对象，而是让客户端构建一个Builder对象（这个Builder对象是它要构建的类的成员内部类），然后在客户端调用Builder对象上的类似于setter的方法，来设置相关的参数。最后通过调用build方法来生成不可变对象。

Builder模式通常做为配置类的构建器将配置的构建与表示分离开来，同时也是将配置从目标类中隔离开来，避免过多的setter方法。

优点：

* 良好的封装性，使用建造者模式可以使客户端不必知道产品内部组成的细节。
* 建造者独立，容易扩展。

缺点：

* 会产生多余的Builder对象以及Director对象，消耗内存。

***

***

***

### 工厂模式

#### 1. 定义

定义一个用于创建对象的借口，让子类决定实例化哪个类。

#### 2. 使用场景

* 任何需要生成复杂对象的的地方，都可以使用工厂模式。
* 复杂对象适合使用工厂模式，用new就可以完成创建的对象无需使用工厂模式。

#### 3. 通用模式代码

```java
// 抽象产品类
public abstract class Product {
    public abstract void method();
}
```

```java
// 具体产品类A
public class ConcreteProductA extends Product {
    @Override
    public void method() {
        // do something
    }
}
```

```java
// 具体产品类B
public class ConcreteProductB extends Product {
    @Override
    public void method() {
        // do something
    }
}
```

```java
// 抽象工厂类
public abstract class Factory {
    public abstract Product createProduct();
}
```

```java
// 具体工厂类A
public class ConcreteFactoryA extends Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductA();
    }
}
```

```java
// 具体工厂类B
public class ConcreteFactoryB extends Factory {
    @Override
    public Product createProduct() {
        return new ConcreteProductB();
    }
}
```

```java
// 使用，通过ConcreteFactoryA的createProduct，就可以创建ConcreteProductA的对象
public static void main(String[] args) {
    Factory factory = new ConcreteFactoryA();
    Product product = factory.createProduct();
    product.method();
}
```

通过工厂对象，来生产对应的产品对象，我们需要哪一个产品就生产哪一个。

针对Factory的升级：

```java
public abstract class Factory {
    public abstract <T extends Product> T createProduct(Class<T> clz);
}
public class ConcreteFactory extends Factory {
    @Override
    public <T extends Product> T createProduct(Class<T> clz) {
        Product product = null;
        try {
            product = (Product) Class.forName(clz.getName()).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return (T) product;
    }
}
// 使用
public static void main(String[] args) {
    Factory factory = new ConcreteFactory();
    Product product = factory.createProduct(ConcreteProductB.class);
    product.method();
}
```

#### 4. 源码中的应用

* Activity的onCreate方法

  ```java
  public class MainActivity extends Activity {
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(new LinearLayout(this));
      }
  }
  ```

  通过new创建一个View对象，并设置给当前界面的ContentView，这就是工厂模式。

#### 5. 总结

优点：

* 降低了对象之间的耦合度。
* 依赖于抽象的架构，其将实例化的任务交由子类去完成，有非常好的扩展性。

缺点：

* 每次添加新产品都需要编写一个新的产品类，同时还要引入抽象层，导致类结构的复杂化。

***

***

***

### 抽象工厂模式

#### 1. 定义

为创建一组相关或者是相互依赖的对象提供一个借口，而不需要指定它们的具体类。

#### 2. 使用场景

一个对象族有相同的约束时可以使用抽象工厂模式。

#### 3. 通用模式代码

```java
// 抽象产品类A
public abstract class AbstractProductA {
    public abstract void method();
}
```

```java
// 抽象产品类B
public abstract class AbstractProductB {
    public abstract void method();
}
```

```java
// 具体产品类A1
public class ConcreteProductA1 extends AbstractProductA {
    @Override
    public void method() {
        // do something
    }
}
```

```java
// 具体产品类A2
public class ConcreteProductA2 extends AbstractProductA {
    @Override
    public void method() {
        // do something
    }
}
```

```java
// 具体产品类B1
public class ConcreteProductB1 extends AbstractProductB {
    @Override
    public void method() {
        // do something
    }
}
```

```java
// 具体产品类B2
public class ConcreteProductB2 extends AbstractProductB {
    @Override
    public void method() {
        // do something
    }
}
```

```java
// 抽象工厂类
public abstract class AbstractFactory {
    public abstract AbstractProductA createProductA();
    public abstract AbstractProductB createProductB();
}
```

```java
// 具体工厂类1
public class ConcreteFactory1 extends AbstractFactory {
    @Override
    public AbstractProductA createProductA() {
        return new ConcreteProductA1();
    }
    @Override
    public AbstractProductB createProductB() {
        return new ConcreteProductB1();
    }
}
```

```java
// 具体工厂类2
public class ConcreteFactory2 extends AbstractFactory {
    @Override
    public AbstractProductA createProductA() {
        return new ConcreteProductA2();
    }
    @Override
    public AbstractProductB createProductB() {
        return new ConcreteProductB2();
    }
}
```

* AbstractFactory：抽象工厂角色，它声明了一组用于创建一种产品的方法，每一个方法对应一种产品。
* ConcreteFactory：具体工厂角色，它实现了在抽象工厂中定义的创建产品的方法，生成一组具体产品，这些产品构成了一个产品的种类，每一个产品都位于某个产品等级结构中。
* AbstractProduct：抽象产品角色，它为每种产品声明接口。
* ConcretePorduct：具体产品角色，它定义具体工厂生产的具体产品对象，实现抽象产品接口中声明的业务方法。

#### 4. 总结

优点：

* 分离接口与实现。

缺点：

* 类文件的爆炸性增加。
* 不太容易扩展新的产品类，因为每当我们增加一个产品类就需要修改抽象工厂，那么所有的具体工厂类均会被修改。

***

***

***

### 策略模式

#### 1. 定义

策略模式定义了一系列的算法，并将每一个算法封装起来，而且使它们还可以互相替换。策略模式让算法独立于使用它的客户而独立变化。

#### 2. 使用场景

* 针对同一类型的问题的多种处理方式，仅仅是具体行为有差别时。
* 需要安全的封装多种同一类型的操作时。
* 出现同一抽象类的多个子类，而又需要使用if-else或者switch-case来选择具体子类时。

#### 3. 简单实现

```java
public class PriceCalculator {
    private static final int BUS = 1;
    private static final int SUBWAY = 2;
    public static void main(String[] args) {
        new PriceCalculator().calculatePrice(16, BUS);
        new PriceCalculator().calculatePrice(16, SUBWAY);
    }
    // 公交车费用计算
    private int busPrice(int km) {
        // 相关算法，并返回最终乘车费用
    }
    // 地铁费用计算
    private int subwayPrice(int km) {
        // 相关算法，并返回最终乘车费用
    }
    private int calculatePrice(int km, int type) {
        if (type == BUS) {
            return busPrice(km);
        } else if (type == SUBWAY) {
            return subwayPrice(km);
        }
        return 0;
    }
}
```

PriceCalculator类很明显并不是单一职责。通过if-else的形式来判断，如果我们增加一种出行方式，如出租车，在或者我们计算方式有变化时，都需要直接修改这个类的代码，它会使代码变得臃肿，难以维护。

接下来用策略模式重构。

```java
// 计算接口
public interface CalculatorStrategy {
    int calculatorPrica(int km);
}
```

```java
// 公交车价格计算策略
public class BusStrategy implements CalculatorStrategy {
    @Override
    public int calculatorPrica(int km) {
        // 公交车费用计算相关算法，并返回最终乘车费用
    }
}
```

```java
// 地铁价格计算策略
public class SubwayStrategy implements CalculatorStrategy {
    @Override
    public int calculatorPrica(int km) {
        // 地铁费用计算相关算法，并返回最终乘车费用
    }
}
```

```java
public class PriceCalculator {
    public static void main(String[] args) {
        PriceCalculator calculator = new PriceCalculator();
        calculator.setStrategy(new BusStrategy());
        int price = calculator.calculatorPrice(16);
    }
    CalculatorStrategy mStrategy;
    public void setStrategy(CalculatorStrategy strategy){
        this.mStrategy = strategy;
    }
    public int calculatorPrice(int km){
        return mStrategy.calculatorPrica(km);
    }
}
```

#### 5. 源码中的实现

* 时间插值器

  动画中的TimeInterpolator，也就是时间插值器，就是策略模式的典型运用。

#### 5. 总结

优点：

* 结构清晰明了，使用简单直观。
* 耦合度相对而言较低，扩展方便。
* 操作封装也更为彻底，数据更为安全。

缺点：

* 随着策略的增加，子类也会变得繁多。

**工厂模式和策略模式的区别**

* 工厂模式中只管生产实例，具体怎么使用工厂实例由调用方决定，策略模式是将生成实例的使用策略放在策略类中配置后才提供调用方使用。 
* 工厂模式调用方可以直接调用工厂实例的方法属性等，策略模式不能直接调用实例的方法属性，需要在策略类中封装策略后调用。
* 工厂模式的作用是创建对象，策略模式的作用是让一个对象选择一个行为。

***

***

***

### 责任链模式

#### 1. 定义

使多个对象都有机会处理请求，从而避免了请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有对象处理它为止。

#### 2. 使用场景

* 多个对象可以处理同一个请求，但具体由哪个对象处理则在运行时动态决定。
* 在请求处理者不明确的情况下向多个对象中的一个提交一个请求。
* 需要动态指定一组对象处理请求。

#### 3. 通用模式代码

```java
// 抽象处理者
public abstract class Handler{
    protected Handler successor; // 下一个节点的处理者
    public abstract void handleRequest(String condition); 
}
```

```java
// 具体的处理者1
public class ConcreteHandler1 extends Handler{
    @Override
    public void handleRequest(String condition) {
        if (condition.equals("ConcreteHandler1")){
            // do something
        } else {
            successor.handleRequest(condition);
        }
    }
}
```

```java
// 具体的处理者2
public class ConcreteHandler2 extends Handler{
    @Override
    public void handleRequest(String condition) {
        if (condition.equals("ConcreteHandler2")){
            // do something
        } else {
            successor.handleRequest(condition);
        }
    }
}
```

```java
public static void main(String[] args) {
    ConcreteHandler1 handler1 = new ConcreteHandler1();
    ConcreteHandler2 handler2 = new ConcreteHandler2();
    handler1.successor = handler2;
    handler2.successor = handler1;
    handler1.handleRequest("ConcreteHandler2");
}
```

#### 4. 源码中的应用

* dispatchTouchEvent

  Android中的事件分发处理。

#### 5. 总结

优点：

* 可以对请求者和处理者关系解耦，提高代码的灵活性。

缺点：

* 对链中请求处理者的遍历，如果处理者太多会影响性能，特别是在一些递归调用中。

***

***

***

### 观察者模式

#### 1. 定义

定义对象间一种一对多的依赖关系，使得每当一个对象改变状态，则所有依赖于它的对象都会得到通知并被自动更新。

#### 2. 使用场景

* 关联行为场景，需要注意的是，关联行为是可拆分的，而不是组合关系；
* 事件多级触发场景；
* 跨系统的消息交换场景，如消息队列、事件总线的处理机制。

#### 3. 简单实现

```java
// 抽象观察者
interface Observer {
    public void update(Observable o, Object content);
}
```

```java
// 具体观察者
class ConcreteObserver implements Observer {
    @Override
    public void update(Observable o, Object content) {
        // 内容更新提示，执行相关操作。
    }
}
```

```java
// 抽象主题。被观察者
interface Observable {
    public void addObserver(Observer o);
    public void removeObserver(Observer o);
    public void notifyObserver(Object content);
}
```

```java
// 具体主题，具体被观察者
class ConcreteObservable implements Observable {
    ArrayList<Observer> mObservers = new ArrayList<>();
    @Override
    public void addObserver(Observer o) {
        mObservers.add(o);
    }
    @Override
    public void removeObserver(Observer o) {
        mObservers.remove(o);
    }
    @Override
    public void notifyObserver(Object content) {
        for (Observer observer : mObservers) {
            observer.update(this, content);
        }
    }
}
```

```java
public static void main(String[] args) {
    Observable observable = new ConcreteObservable();
    observable.addObserver(new ConcreteObserver());
    observable.addObserver(new ConcreteObserver());
}
```

#### 4. 源码中的应用

* Adapter

  ListView添加数据都会调用Adapter的notifyDataSetChanged()方法。

  ```java
  public abstract class BaseAdapter implements ListAdapter, SpinnerAdapter {
      private final DataSetObservable mDataSetObservable = new DataSetObservable();
      public void registerDataSetObserver(DataSetObserver observer) {
          mDataSetObservable.registerObserver(observer);
      }
      public void unregisterDataSetObserver(DataSetObserver observer) {
          mDataSetObservable.unregisterObserver(observer);
      }
      public void notifyDataSetChanged() {
          mDataSetObservable.notifyChanged();
      }
  }
  ```

#### 5. 总结

主要作用：对象解耦，将观察者和被观察者完全隔离，只依赖于Observer和Observable。

优点：

* 观察者和被观察者之间是抽象解耦，应对业务变化；
* 增强系统灵活性、可扩展性。

缺点：

* 一个观察者卡顿，会影响整体的执行效率。所以应该采用异步的方式。

***

***

***

### 代理模式

#### 1. 定义

为其他对象提供一种代理以控制这个对象的访问。

#### 2. 使用场景

当无法或不想直接访问某个对象或访问某个对象存在困难时可以通过一个代理对象来简介访问，为了保证客户端使用的透明性，委托对象和代理对象需要实现相同的接口。

#### 3. 通用模式代码

```java
// 抽象主题类
abstract class Subject {
    public abstract void visit();
}
```

```java
// 真实主题类
class RealSubject extends Subject {
    @Override
    public void visit() {
        // do something
    }
}
```

```java
// 代理类
class ProxySubject extends Subject {
    private Subject mSubject; // 持有真实主题的引用
    public ProxySubject(Subject subject) {
        this.mSubject = subject;
    }
    @Override
    public void visit() {
        // 通过真实主题引用的对象调用真实主题中的逻辑方法
        mSubject.visit();
    }
}
```

```java
public static void main(String[] args) {
    RealSubject real = new RealSubject();
    ProxySubject proxy = new ProxySubject(real);
    proxy.visit();
}
```

#### 4. 源码中的实现

* ActivityManagerProxy

  其具体代理的是ActivityManagerNative的子类ActivityManagerService。

* AIDL

***

***

***

### 装饰模式

#### 1. 定义

动态地给一个对象添加一些额外的职责。就增加功能来说，装饰模式相比生成子类更为灵活。

#### 2. 使用场景

需要透明且动态地扩展类的功能时。

#### 3. 通用模式代码

```java
// 抽象组件类，被装饰的原始对象
abstract class Component {
    public abstract void operate();
}
```

```java
// 组件的具体实现类，装饰的具体对象
class ConcreteComponent extends Component {
    @Override
    public void operate() {
        // do something
    }
}
```

```java
// 抽象装饰者
abstract class Decorator extends Component {
    private Component mComponent;
    public Decorator(Component component) {
        this.mComponent = component;
    }
    @Override
    public void operate() {
        mComponent.operate();
    }
}
```

```java
// 装饰者具体实现类
class ConcreteDecoratorA extends Decorator {
    public ConcreteDecoratorA(Component component) {
        super(component);
    }
    @Override
    public void operate() {
        operateA();
        super.operate();
        operateB();
    }
    // 自定义的装饰A方法
    public void operateA() {
        // do something
    }
    // 自定义的装饰B方法
    public void operateB() {
        // do something
    }
}
```

```java
public static void main(String[] args) {
    Component component = new ConcreteComponent();
    Decorator decorator = new ConcreteDecoratorA(component);
    decorator.operate();
}
```

#### 4. 源码中的应用

* Context

  Context是抽象类，具体实现类ContextImpl，装饰类是ContextWrapper。

#### 5. 总结

优点：

* 装饰模式和继承目的都是扩展对象的功能，装饰模式比继承更灵活。

缺点：

* 类变多，过度使用，程序会变得复杂。

装饰模式和代理模式的区别：

* 装饰模式是以对客户端透明的方式扩展对象的功能，是继承关系的一个替代方案；而代理模式则是给一个对象提供一个代理对象，并由代理对象来控制原有对象的引用。
* 装饰模式应该为所装饰的对象增强功能；代理模式对代理的对象施加控制，但不对对象本身的功能进行增强。

***

***

***

### 参考文献

1. Effective Java 中文第二版

