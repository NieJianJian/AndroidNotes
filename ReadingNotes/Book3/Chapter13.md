## 第13章 热修复原理

### 1 资源修复

### 1.1 Instant Run概述

Instant Run的构建和部署都是基于更改的部分。部署方式有三种，无论那种方式，**都不需要重新安装App**。

* **Hot Swap**：不需要重启App，不需要重启当前Activity。**修改一个现有方法中的代码时会采用**。
* **Warn Swap**：不需要重启App，需要重启Activity。**修改或删除一个现有的资源文件时会采用**。
* **Cold Swap**：需要重启App，但不需要重新安装。**添加、删除或修改一个字段和方法、添加一个类等**。

### 1.2 Instant Run的资源修复

Instant Run资源修复的核心逻辑在MonkeyPatcher的[`monkeyPatchExistingResources`](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book3/source_code/Method_monkeyPatchExistingresources.md)方法中。

Instant Run中 的资源热修复可以简单概括为以下两个步骤：

* 创建新的AssetManager，通过反射调用addAssetPath方法加载外部的资源，这样新创建的AssetManager就含有了外部资源。
* 将AssetManager类型的`mAssets`字段的引用全部替换为新创建的AssetManager。

***

### 2 代码修复

代码修复主要有3个方案，分别是底层替换、类加载和Instant Run。

### 2.1 类加载方案

类加载方案基于Dex分包方案。

* **65536限制**：原因是DVM Bytecode的限制，DVM指令集的方法调用指令`invoke-kind`索引为16bits，最多能引用65535个方法。
* **LinearAlloc限制**：安装应用时可能会提示`INSTALL_FAILED_DEXOPT`，原因是LinearAlloc限制，DVM中的LinearAlloc时一个固定的缓存区，当方法数超出了缓存区的大小时会报错。

为了解决65536限制和LinearAlloc限制，产生了Dex分包方案。

> Dex分包方案主要做的是在打包时将应用代码分为多个Dex，将应用启动时需要用到的类和这些类的直接引用类放到主Dex中，其他代码放到次dex中。当应用启动时先加载主Dex，等到应用启动后再动态地加载次Dex。从而缓解了主Dex的65536限制和LinearAlloc限制。

ClassLoader的加载过程，其中一个缓解就是调用DexPathList的`findClass`方法，如下：

```java
public Class<?> findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) { // 1
        Class<?> clazz = element.findClass(name, definingContext, suppressed); // 2
        if (clazz != null) {
            return clazz;
        }
    }
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}
```

**原理**：Element内部封装了DexFile，DexFile用于加载dex文件，因此每个dex文件对应一个Element。多个Element组成了有序的Element数组dexElements。当要查找类时，会在注释1处遍历Element数组dexElements（相当于遍历dex文件数组），注释2处调用Element的findClass方法，其方法内部会调用DexFile的loadClassBinaryName方法查找类。如果在Element中（dex文件）找到了该类就返回，如果没有找到就在下一个Element中进行查找。根据查找流程，我们将有Bug的类Key.class进行修改，再将Key.class打包成包含dex的补丁包Patch.jar，放在Element数组dexElements的第一个元素，这样会首先找到Patch.dex中的Key.class，排在数组后面的dex文件中存在Bug的Key.class根据ClassLoader的双亲委托模式就不会被加载。这就是类加载方案。

### 2.2 底层替换方案

底层替换方案和反射的原理有些关联，假设我们要反射Key的show方法，会调用如下代码：

```java
Key.class.getDeclaredMethod("show").invoke(Key.class.newInstance());
```

`invoke`方法是一个native方法，对应JNI层的代码为是`java_lang_reflect_Method.cc`里的`Method_invoke`方法，其方法内部又调用到了`reflection.cc`内的`InvokeMethod`方法。该方法中有如下一行代码：

```c++
ObjPtr<mirror::Executable> executable = soa.Decode<mirror::Executable>(javaMethod);
ArtMethod* m = executable->GetArtMethod(); // 1
```

注释1处获取传入的`javaMethod`（key的show方法）在ART虚拟机中对应的一个ArtMethod指针。

ArtMethod结构体中包含了Java方法所有信息，包括执行入口、访问权限、所属类和执行地址等。

在[ArtMethod](https://github.com/NieJianJian/AndroidNotes/blob/master/ReadingNotes/Book2/source_code/art_method.h.md)结构中比较重要的字段是`dex_cache_resolved_methods`和`entry_ponit_from_quick_compiled_code`，它们是方法的执行入口，当我们调用某一个方法时（比如Key的show方法），就会获取到show方法的执行入口，通过执行入口就可以跳过去执行show方法。

**替换ArtMethod结构体中的字段或者替换整个ArtMethod结构体，这就是底层替换方案**。

* AndFix采用替换结构体中的字段，由于厂商可能会修改ArtMethod结构体，所以可能会导致替换失败。
* Sophix采用替换整个ArtMethod结构体，所以不存在兼容问题。

### 2.3 Instant Run方案

**ASM**：是一个Java字节码操控框架，它能够动态生成类或者增强现有类的功能。ASM可以直接产生class文件，也可以在类加载到虚拟机之前动态改变类的行为。

**原理**：Instant Run在第一次构建APK时，使用ASM在每一个方法中注入了类似如下的代码：

```java
IncrementalChange localIncrementalChange = $change; // 1
if (localIncrementalChange != null) { // 2
    localIncrementalChange.access$dispatch("onCreate.(Landroid/os/Bundle;)V", 
        new Object[] {this, paramBundle});
    return;
}
```

* 注释1处是一个成员变量`localIncrementalChange`，它的值为`$change`。

* `$change`实现了IncrementalChange接口。

* 点击Instant Run时，如果方法没有变化则`$change`为null，就调用return，不做任何处理

* 如果方法有变化，就生成替换类。

  假设MainActivity的`onCreate`方法做了修改，就会生成替换类`MainActivity$override`，这个类实现了IncrementalChange接口，同时会生成一个AppPatchedLoaderImpl类，这个类的`getPatchedClasses`方法会返回被修改的类的列表（里面包含MainActivity），根据列表会将MainAcitivyt的`$change`设置为`MainActivity$override`，因此满足注释2的条件，会执行`MainActivity$override`的`access$diapatch`方法，在`access$dispatch`方法中会根据参数"onCreate.(Landroid/os/Bundle;)V"执行`MainActivity$override`的`onCreate`方法。从而实现了`onCreate`方法的修改。

***

### 3 动态链接库的修复

so修复主要有两个方案：

* 将so补丁插入到NativeLibraryElement数组的前部，让so补丁的路径先返回和加载。
* 调用System的load方法来接管so的加载入口。