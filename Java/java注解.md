## java注解

### 1. 注解分类

注解分为标准注解和元注解。

#### 1.1 标准注解

标准注解有以下四种。

* @Override：对覆盖超类中的方法进行标记，如果被标记的方法并没有实际覆盖超类中的方法，则编译器将会发出错误警告。
* @Deprecated：对不鼓励使用或者已过时的方法添加注解，当编程人员使用这些方法时，将会在编译时显示提示信息。
* @SuppressWarnings：选择性的取消特定代码段中的警告。
* @SafeVarargs：JDK 7 新增，用来生命使用了可变长度参数的方法，其在与泛型累一起使用时不会出现类型安全问题。

#### 1.2 元注解

元注解，用来注解其他注解，从而创建新的注解，有以下几种。

* @Target：注解所修饰的对象范围。
* @Inherited：表示注解可以被继承。
* @Documented：表示这个注解应该被JavaDoc工具记录。
* @Retention：用来声明注解的保留策略。
* @Repeatable：JDK 8 新增，允许一个注解在同一个声明类型（类、属性或方法）上多次使用。

其中@Target注解取值是一个ElementType类型的数组，几种取值如下：

* ElementType.TYPE：能修饰类、接口或枚举类型。
* ElementType.FIELD：能修饰成员变量。
* ElementType.METHOD：能修饰方法。
* ElementType.PARAMETER：能修饰参数。
* ElementType.CONSTRUCTOR：能修饰构造方法。
* ElementType.LOCAL_VARIABLE：能修饰局部变量。
* ElementType.ANNOTATION_TYPE：能修饰注解。
* ElementType.PACKAGE：能修饰包。
* ElementType.TYPE_PARAMETER：类型参数声明。
* ElementType.TYPE_USE：使用类型。

其中@Retention注解有3种类型，分别表示不同级别的保留策略。

* RetentionPolicy.SOURCE：源码级注解。注解只会保留在.java源码中，源码在编译后，注解信息被丢弃，不会保留在.class文件中。
* RetentionPolicy.CLASS：编译时注解。注解信息会保留在.java源码以及.class中。当运行Java程序时，JVM会丢弃该注解，不会保留在JVM中。
* RetentionPolicy.RUNTIME：运行时注解。当运行Java程序时，JVM也会保留该注解信息，可以通过反射获取该注解信息。

***

### 2. 定义注解

#### 2.1 基本定义

定义新的注解类型需要使用@interface关键字，这与定义一个接口很像，如下：

```java
public @interface Swordsman{
...  
}
```

定义完注解后，就可以在程序中使用该注解：

```java
@Swordsman
public class AnnotationTest{
...
}
```

#### 2.2 定义成员变量

注解只有成员变量，没有方法。

* 注解的成员变量在注解定义中以"无形参的方法"形式来声明；
* 其"方法名"定义了该成员变量的名字；
* 其返回值定义了该成员变量的类型。

```java
public @interface Swordsman {
    String name();
    int age();
}
```

使用注解时应该为该注解的成员变量指定值。

```java
public class AnnotationTest {
    @Swordsman(name = "张无忌", age = 30)
    public void fighting() {
        ...
    }
}
```

也可在定义注解的成员变量时，使用**default关键字**为其指定默认值，如下所示：

```java
public @interface Swordsman {
    String name() default "张无忌";
    int age() default 23;
}
```

注解定义了默认值，使用的时候，就不用为其指定直了，直接使用默认值：

```java
@Swordsman
public void fighting() {
    ...
}
```

#### 2.3 限定参数的值和范围

@IntDef：限定取值为int类型。

@StringDef：限定取值为String类型。

@IntRange：限定取值范围

**以上几种注释，都是Android API中的，Java中并没有**。

```java
public static final int VERTICAL = 0;
public static final int HORIZONTAL = 1;

@IntDef({VERTICAL, HORIZONTAL})
public @interface OrientationType {
}

/**
 * @param orientation 只能输入 VERTICAL 和 HORIZONTAL
 * @param count 只能输入0-10的值。
 */
public static void setOrientation(@OrientationType int orientation,
                                  @IntRange(from = 0, to = 10) int count) {
    // do something...
}
```

#### 2.4 定义运行时注解

可以用@Retention来设定注解的保留策略。

这3种策略的生命周期长度：SOURCE < CLASS < RUNTIME。

* 如果需要在运行时去动态获取注解信息，那只能用RetentionPolicy.RUNTIME；
* 如果要在编译时进行一些预操作，比如生成一些辅助代码，用RetentionPolicy.CLASS；
* 如果只是做一些检查性的操作，比如@Override，则可用RetentionPolciy.SOURCE。

定义运行时注解方法如下：

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface Swordsman {
    String name() default "张无忌";
    int age() default 23;
}
```

***

### 3. 注解处理器

[注解处理器](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/AnnotationProcessor.md)

