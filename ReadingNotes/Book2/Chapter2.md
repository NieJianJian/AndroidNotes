## 第2章 热替换代码修复

### 1.底层热替换原理

### 1.1 Andfix回顾

**Q：**为何唯独Andfix能够做到即时生效？

**A：**在App启动到一半时，所需发生变更的分类已经被加载过了，在**Android系统中时无法对一个分类进行卸载的。**而腾讯洗的方案是让Classloader去加载新的类，如果不重启App，原有的类还在虚拟机中，就无法加载新类。因此，只有在下重启是，还没有运行到业务逻辑之前抢先加载补丁中的新类。

　　**Andfix是直接在已经加载的类中的native层替换掉原有方法，是在所有类的基础上进行修改。**

　　**Android 4.4以下版本用Dalvik虚拟机，4.4以上用的是Art虚拟机。**

　　**每一个Java方法在Art虚拟机中都对应一个ArtMethod，ArtMethod记录了这个Java方法的所有信息，包括所属类、访问权限、代码执行地址等。**（art/runtime/art_method.h）

　　**Andfix执行过程：**通过env->FromReflectedMethod，可以由Method对象得到这个方法所对应的ArtMethod的真正起始地址，然后强制转换成ArtMethod指针，将旧函数的所有成员变量替换为新函数的。

### 1.2 虚拟机调用方法的原理

