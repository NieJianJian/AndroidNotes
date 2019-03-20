## Java优化性能的45个细节

***ImportNew -> 来源：网络，原始作者未知。如有知晓的朋友，请留言。***

​	在JAVA程序中，性能问题的大部分原因并不在于JAVA语言，而是程序本身。养成良好的编码习惯非常重要，能够显著地提升程序性能。



### 1.尽量在合适的场合使用单例

​	使用单例可以减轻加载的负担，缩短加载的时间，提高加载的效率，但并不是所有地方都适用于单例，简单来说，单例主要适用于以下三个方面：

* 控制资源的使用，通过线程同步来控制资源的并发访问；
* 控制实例的产生，以达到节约资源的目的；
* 控制数据共享，在不建立直接关联的条件下，让多个不相关的进程或线程之间实现通信。

***



### 2.尽量避免随意使用静态变量

当某个对象被定义为static变量所引用，那么GC通常是不会回收这个对象所占有的内存，如：

```java
public class A{
    private static B b = new B();
}
```

此时静态变量b的生命周期与A类同步，如果A类不会卸载，那么b对象会常驻内存，直到程序终止。

***



### 3.尽量避免过多过常地创建Java对象

​	尽量避免在经常调用的方法，循环中new对象，由于系统不仅要花费时间来创建对象，而且还要花时间对这些对象进行垃圾回收和处理，在我们可以控制的范围内，最大限度地重用对象，最好能用基本的数据类型或数组来替代对象。

***

### 4. 尽量使用final修饰符

​	带有final修饰符的类是不可派生的。在JAVA核心API中，有许多应用final的例子，例如java、lang、String，为String类指定final防止了使用者覆盖length()方法。另外，如果一个类是final的，则该类所有方法都是final的。java编译器会寻找机会内联（inline）所有的final方法（这和具体的编译器实现有关），此举能够使性能平均提高50%。

如：让访问实例内变量的getter/setter方法变成”final：

简单的getter/setter方法应该被置成final，这会告诉编译器，这个方法不会被重载，所以，可以变成”inlined”,例子：

```java
class MAF {
  
		public void setSize (int size) {
				_size = size;
		}
	
		private int _size;
}

更正

class DAF_fixed {

		final public void setSize (int size) {
				_size = size;
		}

		private int _size;
}
```









