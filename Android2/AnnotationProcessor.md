## 注解处理器

[Java注解](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/java注解.md)

对于不同的注解有不同的注解处理器。

* 针对运行时注解，采用反射机制处理。
* 针对编译时注解，采用AbstractProcessor处理。

### 1. 运行时注解处理器

首先定义运行时注解：

```java
@Documented
@Target(METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface GET {
    String value() default "";
}
```

上面的代码是Retrofit中定义的@GET注解，作用于方法。接下来应用该注解，如下所示：

```java
public class AnnotationTest {
    @GET(value = "http://ip.taobao.com/59.108.43.37")
    public void getIpMsg() {
        return "";
    }
    @GET(value = "http://ip.taobao.com/")
    public void getIp() {
        return "";
    }
}
```

接下来写一个注解处理器，如下所示：

```java
public class AnnotationTest {
    public static void main(String[] args) {
        Method[] methods = AnnotationTest.class.getDeclaredMethods();
        for (Method m : methods) {
            GET get = m.getAnnotation(GET.class);
            System.out.println(get.value);
        }
    }
}
```

上面代码用到了两个反射方法：getDeclaredMethods和getAnnotation，它们都属于AnnotationElement接口，Class、Method和Field等类都实现了该接口。

调用getAnnotation方法返回指定类型的注解对象，也就是GET。

调用GET的value方法返回从GET对象中提取元素的值。

***

### 2. 编译时注解处理器

1. 定义注解

   新建一个Java Library来专门存放注解，命名为annotations。接下来定义注解，如下所示：

   ```java
   @Retention(RetentionPolicy.CLASS)
   @Target(ElementType.FIELD)
   public @interface BindView {
       int value() default 1;
   }
   ```

2. 编写注解处理器

   在项目中新建一个Java Library来存放注解处理器，名为processor，配置它的build.gradle：

   ```java
   apply plugin: 'java-library'
   
   dependencies {
       implementation fileTree(dir: 'libs', include: ['*.jar'])
       implementation project(':annotations')
   }
   
   sourceCompatibility = "7"
   targetCompatibility = "7"
   ```

   接下来编写注解处理器ClassProcessor，它继承AbstractProcessor，如下所示：

   ```java
   public class ClassProcessor extends AbstractProcessor {
       @Override
       public synchronized void init(ProcessingEnvironment processingEnvironment) {
           super.init(processingEnvironment);
       }
       @Override
       public boolean process(Set<? extends TypeElement> set,
                              RoundEnvironment roundEnvironment) {
           ...
           return true;
       }
       @Override
       public Set<String> getSupportedAnnotationTypes() {
           Set<String> annotations = new LinkedHashSet<>();
           annotations.add(BindView.class.getCanonicalName());
           return annotations;
       }
       @Override
       public SourceVersion getSupportedSourceVersion() {
           return SourceVersion.latestSupported();
       }
   }
   ```

   * init：被注解处理工具调用，并输入ProcessingEnvironment参数。ProcessingEnvironment提供很多有用的工具类，如Elements、Types、Filer和Messager等。
   * process：相当于每个处理器的主函数main()，在这里写你的扫描、评估和处理注解的代码，以及生成Java文件。输入参数RoundEnviroment，可以让你查询出包含特定注解的被注解元素。
   * getSupportedAnnotationTypes：这是必须指定的方法，指定这个注解处理器是注册给哪个注解的。注意，它的返回值是一个字符串的集合，包含本处理器想要处理的注解类型的合法全称。
   * getSupportedSourceVersion：用来指定你使用的Java版本，这里通常返回latestSupperted()。

   Java 7 以后可以使用注解来代替getSupportedAnnotationTypes方法和getSupportedSourceVersion方法:

   ```java
   @SupportedSourceVersion(SourceVersion.RELEASE_8)
   @SupportedAnnotationTypes("com.nj.annotation.BindView")
   public class ClassProcessor extends AbstractProcessor {
       @Override
       public synchronized void init(ProcessingEnvironment processingEnvironment) {
           super.init(processingEnvironment);
       }
       @Override
       public boolean process(Set<? extends TypeElement> set,
                              RoundEnvironment roundEnvironment) {
           ...
           return true;
       }
   }
   ```

   但是考虑到Android兼容性的问题，这里不建议采用这种注解方式。

   接下来编写process方法，如下所示：

   ```java
   public boolean process(Set<? extends TypeElement> set,
                          RoundEnvironment roundEnvironment) {
       Messager messager = processingEnv.getMessager();
       for (Element element : roundEnvironment.getElementsAnnotatedWith(BindView.class)) {
           if (element.getKind() == ElementKind.FIELD) {
               messager.printMessage(Diagnostic.Kind.NOTE,
                       "printMessage:" + element.toString());
           }
       }
       return true;
   }
   ```

   这里用Messager的printMessage方法打印出注解修饰的成员变量的名称，这个名称会在Android Studio的Gradle Console窗口打印出。

3. 注册注解处理器

   为了能使用注解处理器，需要用一个服务文件来注册它。

   * 首先在processor库的main目录下新建resources资源文件夹；
   * 接下来在resources中再建立META-INF/services目录文件夹；
   * 最后在META-INF/services中创建javax.annotation.processing.Processor文件，这个文件中的内容是注解处理器的名称。本例中的内容为：com.nj.processor.ClassProcessor。

   如果觉得创建服务文件比较麻烦，可以使用开源项目AutoService，它可以帮助我们完成上述过程。

   * 首先添加该库

     ```java
     dependencies {
         ...
         implementation 'com.google.auto.service:auto-service:1.0-rc2'
     }
     ```

   * 最后在注解处理器ClassProcessor中添加@AutoService即可：

     ```java
     @AutoService(Processor.class)
     public class ClassProcessor extends AbstractProcessor {
         ...
     }
     ```

4. 应用注解

   接下来在我们主工程项目(app)中引用注解。

   首先在build.gradle中引用annotations和processor这两个库：

   ```java
   dependencies {
       ...
       implementation project(':annotations')
       implementation project(':processor')
   }
   ```

   接下来在MainActivity中应用注解：

   ```java
   public class MainActivity extends Activity {
       @BindView(value = R.id.textview1)
       TextView mTextView;
       @Override
       protected void onCreate(Bundle savedInstanceState) {
           super.onCreate(savedInstanceState);
           setContentView(R.layout.activity_main);
       }
   }
   ```

   最后Rebuild Project，在Gradle Console窗口打印结果如下所示：

   ```java
   ...
   注: printMessage:mTextView
   
   > Task :app:compileDebugNdk NO-SOURCE
   ...
   ```

   可以发现编译时打印出了@BindView注解修饰的成员变量名：mTextView。

5. 使用android-apt插件

   我们主项目（app）中引用了processor库，但注解处理器只在编译处理期间需要用到，编译处理完后就没有实际作用了，而主工程项目中添加了这个库会引入很多不必要的文件。为了处理这个问题，我们引入了插件android-apt。它的主要作用有两个：

   * 仅仅在编译期间去依赖注解处理器所在的函数库并进行工作，但不会打包到APK中。
   * 为注解处理器生成的代码设置好路径，以便Android Studio能找到它。

   接下来介绍如何使用它。首先在整个工程（Project）的build.gradle中添加如下代码：

   ```java
   buildscript {
       dependencies {
           classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
           ...
       }
   }
   ```

   然后在主工程项目（app）的build.gradle中以apt的方式引入注解处理器processor，如下：

   ```java
   ...
   apply plugin: 'com.neenbedankt.android-apt'
   ...
   dependencies {
       ...
       // implementation project(':processor')
       apt project(':processor')
   }
   ```

   