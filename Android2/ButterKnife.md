## ButterKnife原理

了解ButterKnife原理需要的基础知识：

* [Java注解](https://github.com/NieJianJian/AndroidNotes/blob/master/Java/java注解.md)
* [注解处理器](https://github.com/NieJianJian/AndroidNotes/blob/master/Android2/AnnotationProcessor.md)

### 1. 依赖注入原理

了解ButterKnife之前，我们先来讲讲依赖注入。我们通过控制反转来更好的了解依赖注入。

* 控制反转

  多个对象之间的依赖关系复杂，耦合严重。为了解决对象之间的耦合度过高的问题，提出了IoC理论，用来实现对象之间的解耦。

  IoC 是 Inversion of Control 的缩写，即控制反转。

  IoC 理论提出的观点大致是这样的：借助于"第三方"实现具有依赖关系的对象之间的解耦。

  假设有四个对象A、B、C、D，它们之间互相耦合，然后引入IoC容器，使得A、B、C、D四个对象不再耦合，它们都依靠于IoC容器。

  在没有引入IoC容器之前，对象A依赖于对象B，那么对象A在初始化或者运行到某一点时，自己必须主动去创建对象B或者使用已经创建的对象B。无论是创建对象B还是使用对象B，控制权都在自己手上。当引入IoC容器之后，对象A和对象B之间失去了联系，当对象A运行到需要对象B时，IoC容器会主动创建一个对象B注入到对象A需要的地方。通过引入Ioc容器前后的对比，可以看出：对象A获取对象B的过程，由主动变成了被动，控制权颠倒了，这就是**控制反转名称的由来**。

* 依赖注入

  控制反转是"哪些方面的控制被反转了"呢？答案是"获取依赖对象的过程被反转了"。于是控制反转起了一个更合适的名字**依赖注入**。所谓依赖注入，是指由IoC容器在运行期间，动态地将某种依赖关系注入到对象中。

Android目前主流的依赖注入框架有ButterKnife和Dagger2。

* ButterKnife从严格意义上将，不算是依赖注入框架，它只是专注于Android系统的View注入框架，并不支持其他方面的注入。
* Dagger2是一个基于JSR-330（Java依赖注入）标准的依赖注入框架，在编译期间自动生成代码，负责依赖对象的创建。

### 2. ButterKnife

