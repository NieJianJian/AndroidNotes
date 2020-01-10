## Java的自定义ClassLoader

**局限性**：系统提供的类加载器只能够加载指定目录下的jar包和Class文件。

**使用场景**：如果想要加载网络上或者D盘某一文件中的jar包和Clas文件则需要自定义ClassLoader。

**操作步骤**：自定义ClassLoader需要如下两个步骤：

* (1)定义一个自定义ClassLoader并继承抽象类ClassLoader。
* (2)复写`findClass`方法，并在`findClass`方法中调用`defineClass`方法。

***

**案例**：自定义一个ClassLoader用来加载位于`D:lib`的Class文件。

* 编写测试类并生成Class文件，代码如下：

  ```java
  package com.example;
  public class Jobs {
  	public void say() {
  		System.out.println("One more thing");
  	}
  }
  ```

* 将编写好的`Jobs.java`文件放入`D:\lib`中。

* 使用cmd命令进入`D:\lib`目录中，执行`javac Jobs.java`命令，生成`Jobs.class`文件。

* 在AS中创建一个Java Library，编写自定义ClassLoader类，代码如下：

  ```java
  import java.io.ByteArrayOutputStream;
  import java.io.File;
  import java.io.FileInputStream;
  import java.io.IOException;
  import java.io.InputStream;
  
  public class DiskClassLoader extends ClassLoader {
      private String path;
  
      public DiskClassLoader(String path) {
          this.path = path;
      }
  
      @Override
      protected Class<?> findClass(String name) throws ClassNotFoundException {
          Class clazz = null;
          byte[] classData = loadClassData(name); // 1
          if (classData == null) {
              throw new ClassNotFoundException();
          } else {
              clazz = defineClass(name, classData, 0, classData.length); // 2
          }
          return clazz;
      }
  
      private byte[] loadClassData(String name) {
          String fileName = getFileName(name);
          File file = new File(path, fileName);
          InputStream in = null;
          ByteArrayOutputStream out = null;
          try {
              in = new FileInputStream(file);
              out = new ByteArrayOutputStream();
              byte[] buffer = new byte[1024];
              int length = 0;
              while ((length = in.read(buffer)) != -1) {
                  out.write(buffer, 0, length);
              }
              return out.toByteArray();
          } catch (IOException e) {
              e.printStackTrace();
          } finally {
              try {
                  if (in != null) {
                      in.close();
                  }
              } catch (IOException e) {
                  e.printStackTrace();
              }
              try {
                  if (out != null) {
                      out.close();
                  }
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
          return null;
      }
  
      private String getFileName(String name) {
          int index = name.lastIndexOf('.');
          if (index == -1) {//如果没有找到'.'则直接在末尾添加.class
              return name + ".class";
          } else {
              return name.substring(index + 1) + ".class";
          }
      }
  }
  ```

  * 注释1处的`loadClassData`方法会获取class文件的字节码数组
  * 注释2处调用`defineClass`方法将class文件的字节码数组转为Class类的实例。
  * 在`loadClassData`方法中需要对流进行操作，关闭流的操作要放在finally语句块中，并且要对in和out分别使用try语句，如果in和out共同再一个try语句中，假设`in.close()`发生异常的话，就无法执行`out.close()`。

* 最后来验证DiskClassLoader是否可以用，代码如下：

  ```java
  package com.nfkj.classloadertest;
  
  import java.lang.reflect.InvocationTargetException;
  import java.lang.reflect.Method;
  
  public class ClassLoaderTest {
      public static void main(String[] args) {
          DiskClassLoader diskClassLoader = new DiskClassLoader("D:\\lib"); // 1
          try {
              Class c = diskClassLoader.loadClass("com.example.Jobs"); // 2
              if (c != null) {
                  try {
                      Object obj = c.newInstance();
                      System.out.println(obj.getClass().getClassLoader());
                      Method method = c.getDeclaredMethod("say", null);
                      method.invoke(obj, null);//3
                  } catch (InstantiationException | IllegalAccessException
                          | NoSuchMethodException
                          | SecurityException |
                          IllegalArgumentException |
                          InvocationTargetException e) {
                      e.printStackTrace();
                  }
              }
          } catch (ClassNotFoundException e) {
              e.printStackTrace();
          }
      }
  }
  ```

  * 注释1处创建`DiskClassLoader`并传入要加载类的路径。

  * 注释2处加载Class文件

  * 注释3处通过反射调用`Jobs`的`say`方法，打印结果如下：

    ```
    com.example.DiskClassLoader@677327b6 One more thing
    ```

  **注**：不要在项目工程中存在名为`com.example.Jobs`的java文件，否则就不会使用`DiskClassLoader`来加载，而是使用`AppClassLoader`来负责加载。