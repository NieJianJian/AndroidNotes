## Java反射

### 1.Class 类

* **类是对象，类是java.lang.Class类的实例对象**

  ```java
  public class MyClass {
      public static void main(String[] args) {
          // People的实例对象
          People people = new People();
          // 任何一个类，都是java.lang.Class的实例对象
      }
  }
  // People本身就是一个对象，是java.lang.Class的对象
  class People { }
  ```

* **Class类的三种表达方式**

  ```java
  Class c1 = People.class;
  // 方法2：通过已知类对象的getClass方法
  Class c2 = people.getClass();
  // 方法3：通过Class类静态方法forName()
  Class c3 = Class.forName("com.nj.reflect.People");
  
  /*
   * c1、c2表示了People类的类类型（class type）
   * 类也是对象，是Class类的实例对象，这个对象我们称为该类的类类型
   */
  // 不管是c1、c2还是c3都代表People的类类型，一个类只可能是Class类的一个实例对象。
  System.out.print(c1 == c2); // true
  System.out.print(c1 == c3); // true
  ```

* **获取对象实例**

  ```java
  // 通过People类的类类型c1、c2或c3创建该类的实例对象。
  People people1 = (People) c1.newInstance(); // 需要有无参构造方法
  ```

  

### 2.动态加载类

* **Class.forName("类的全称")**

  * 不仅表示了类的类类型，还代表了动态加载类
  * 编译时刻加载类是静态加载类，运行时刻加载类是动态加载类。

* **静态加载类**

  ```java
  public class Animal {
      public static void main(String[] args) {
          if (args[0].equals("Dog")) {
              Dog dog = new Dog();
              dog.eat();
          }
          if (args[0].equals("Cat")) {
              Cat cat = new Cat();
              cat.eat();
          }
      }
  }
  class Dog {
      public void eat() { }
  }
  class Cat {
      public void eat() { }
  }
  ```

  * 静态加载类的缺点

    1.如果Dog或者Cat不存在，会报错。但是Dog和Cat不一定会使用。

    2.通过new创建对象是静态加载类，在编译时刻，就需要加载所有可能使用到的类。

* **动态加载类**

  通过Class.forName()动态加载类，就不会在编译时报错了。

  ```java
  public class AnimalManager {
      public static void main(String[] args){
          try {
              // 动态加载类，在运行时刻加载。编译不会再报错。
              Class c = Class.forName(args[0]);
              // 创建对象，通过类类型创建该类的对象
              Object o = c.newInstance();
          } catch (ClassNotFoundException e) {
              e.printStackTrace();
          }
      }
  }
  ```

  但是我们通过c.newInstance()创建类对象的时候，问题来了，是该转换成Dog，还是该转换成Cat。所以，需要统一标准，创建Animals接口，使Dog和Cat都实现Animals接口。

  ```java
  public class AnimalManager {
      public static void main(String[] args){
          try {
              // 动态加载类，在运行时刻加载。编译不会再报错。
              Class c = Class.forName(args[0]);
              // 创建对象，通过类类型创建该类的对象
              Animals animals = (Animals) c.newInstance();
              animals.eat();
          } catch (Exception e) {
              e.printStackTrace();
          }
      }
  }
  interface Animals{
      void eat();
  }
  class Dog implements Animals{
      @Override
      public void eat() {}
  }
  class Cat implements Animals{
      @Override
      public void eat() {}
  }
  ```

  * 优点

    1.动态加载，更加灵活

    2.方便拓展，如果想要加个Pig类，直接创建Pig类，实现Animals接口和方法即可，无需再修改AnimalManager类。



### 3.反射的使用

* **获取class对象的相关信息**

  1.获取class对象的成员变量

  ```java
  // 获取class1对象的所有成员变量，不包括父类的成员变量
  Field[] allFields = class1.getDeclaredFields();
  // 获取class1对象的public成员变量，包括父类里面的成员变量
  Field[] publicFields = class1.getFields();
  // 获取class1对象指定的成员变量，父类的无法获取
  Field name1 = class1.getDeclaredField("name");
  // 获取class1对象指定的public成员变量，包括查找父类
  Field name2 = class1.getField("name");
  
  // 获取变量的名称
  String name = field.getName();
  // 获取变量的类型
  Class type = field.getType();
  ```

  2.获取class对象的方法

  ```java
  // 获取class1对象的所有方法，不包括父类的方法
  Method[] allMethods = class1.getDeclaredMethods();
  // 获取class1对象的public方法，包括父类的public方法
  Method[] publicMethods = class1.getMethods();
  // 获取class1对象指定的方法，父类的无法获取
  Method method1 = class1.getDeclaredMethod("getName");
  // 获取class1对象指定的方法，并且有一个int类型参数的方法
  Method method2 = class1.getDeclaredMethod("getName", int.class);
  // 获取class1对象有两个参数的指定方法
  Method method3 = class1.getDeclaredMethod("getName", int.class, String.class);
  // 获取class1对象指定的public方法，包括父类的public方法。
  Method method4 = class1.getMethod("getName");
  ```

  3.获取class对象的构造函数

  ```java
  // 获取class1对象所有的构造函数
  Constructor[] allConstructors = class1.getDeclaredConstructors();
  // 获取class1对象public的构造函数
  Constructor[] publicConstructors = class1.getConstructors();
  // 获取class1对象指定参数的构造函数
  Constructor constructor1 = class1.getDeclaredConstructor(int.class);
  // 获取class1对象指定参数并且为public的构造函数
  Constructor constructor2 = class1.getConstructor(int.class, String.class);
  ```

  4.class对象的其他方法

  ```java
  Annotation[] annotations = class1.getAnnotations();// 获取class对象的所有注解
  Annotation annotation = class1.getAnnotation(Override.class);// 获取class对象指定注解
  Class superclass = class1.getSuperclass(); // 获取class对象直接父类的类对象
  Type genericSuperclass = class1.getGenericSuperclass(); // 获取class对象的直接父类的Type
  Type[] genericInterfaces = class1.getGenericInterfaces(); // 获取class对象所有接口的Type集合
  String name = class1.getName(); // 获取class对象名称，包括报名路径
  String simpleName = class1.getSimpleName(); // 获取class类名
  Package aPackage = class1.getPackage(); // 获取class包信息
  boolean annotation = class1.isAnnotation();// 判断是否为注解类
  boolean anEnum = class1.isEnum();// 判断是否为枚举
  boolean anInterface = class1.isInterface();// 判断是否为接口类
  boolean array = class1.isArray();//判断是否为集合类型
  boolean primitive = class1.isPrimitive(); // 判断是否为原始类型（8个基础类型）
  boolean anonymousClass = class1.isAnonymousClass(); // 判断是否是匿名内部类
  boolean annotation1 = class1.isAnnotationPresent(Override.class);//判断是否被某个注解类修饰
  ```

  5.Method的相关方法

  ```java
  Method method = class1.getMethod("getName", String.class);
  String name = method.getName(); // 获取方法名称
  // 获取方法的返回值类型的类类型
  Class returnType = method.getReturnType();
  // 获取方法的参数列表类型的类类型的数组
  Class[] parameterTypes = method.getParameterTypes();
  ```

* **Java反射的使用**

  1.生成来的实例对象

  ```java
  // 方法一：调用newInstance()方法，但是这个方法要求class1的类必须有无参构造函数。
  Dog dog = (Dog) class1.newInstance();
  // 方法二：通过getConstructor获取有参的Constructor构造函数，再调用newInstance()创建实例
  Constructor constructor = class1.getConstructor(String.class);
  Dog dog1 = (Dog) constructor.newInstance("dog");
  ```

  2.调用类的方法

  ```java
  // 第一步：生成类对象
  Dog dog = (Dog) class1.newInstance();
  // 第二步：通过class对象的getMethod()等方法，获得具体需要执行的Method对象。
  Method method = class1.getDeclaredMethod("getName", String.class);
  // 如果方法是private修饰，直接调用invoke，会报错，Java要求程序必须有调用该方法的权限。
  // 调用setAccessible，并将参数设置为true，就会取消Java语言的访问权限检查。
  method.setAccessible(true);
  // 调用Method对象的invoke方法，第一个参数是调用该方法的实例对象，第二个参数是对应需要传入的参数。
  // invoke的返回值：如果方法的返回值为void，则invoke返回null，如果不是void，则返回具体的返回值
  // 相当于 dog.getName("dog");
  Object o = method.invoke(dog, "dog");
  ```

  3.访问成员变量的值

  ```java
  // 第一步：生成类对象
  Dog dog = (Dog) class1.newInstance();
  // 第二步：通过getDeclaredField等方法，获取相应的Field对象
  Field field = class1.getDeclaredField("age");
  // 取消Java权限检查
  field.setAccessible(true);
  // 获取dog对象中age成员变量的值
  int age = field.getInt(dog);
  // 修改dog对象中age成员变量的值
  field.setInt(dog, 25);
  // 获取dog对象中String类型的成员变量
  Field field1 = class1.getDeclaredField("name");
  // 取消Java权限检查
  field1.setAccessible(true);
  // 获取name变量的值
  Object o = field1.get(dog);
  // 修改name变量的值
  field1.set(dog, "sam");
  ```



**参考资料**

1.[Java反射](https://github.com/LRH1993/android_interview/blob/master/java/basis.md)

