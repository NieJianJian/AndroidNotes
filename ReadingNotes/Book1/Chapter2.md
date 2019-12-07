## 第2章 Java内存区域与内存溢出异常

### 1.Java虚拟机运行时数据区

![](https://github.com/NieJianJian/AndroidNotes/blob/master/Picture/jvmruntimedata.jpg)

#### 1.1 程序计数器

​		是一块较小的内存空间，看作是当前线程所执行的字节码的行号指示器。字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

