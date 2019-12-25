## 虚拟机在dexopt的时候如何找到某个dex的所有类定义

文件：[android-4.4.4-r1/dalvik/vm/analysis/DexPrepare.cpp](https://www.androidos.net.cn/android/4.4.4_r1/xref//dalvik/vm/analysis/DexPrepare.cpp)

```c++
/*
 * Verify and/or optimize all classes that were successfully loaded from
 * this DEX file.
 */
static void verifyAndOptimizeClasses(DexFile* pDexFile, bool doVerify,
    bool doOpt)
{
    u4 count = pDexFile->pHeader->classDefsSize;
    u4 idx;

    for (idx = 0; idx < count; idx++) {
        const DexClassDef* pClassDef;
        const char* classDescriptor;
        ClassObject* clazz;

        pClassDef = dexGetClassDef(pDexFile, idx);
        classDescriptor = dexStringByTypeIdx(pDexFile, pClassDef->classIdx);

        /* all classes are loaded into the bootstrap class loader */
        clazz = dvmLookupClass(classDescriptor, NULL, false);
        if (clazz != NULL) {
            verifyAndOptimizeClass(pDexFile, clazz, pClassDef, doVerify, doOpt);

        } else {
            // TODO: log when in verbose mode
            ALOGV("DexOpt: not optimizing unavailable class '%s'",
                classDescriptor);
        }
    }
}
```

正是`dexGetClassDef`函数返回了类的定义。

文件：[android-4.4.4_r1/dalvik/libdex/DexFile.h](https://www.androidos.net.cn/android/4.4.4_r1/xref//dalvik/libdex/DexFile.h)

```c++
/* return the ClassDef with the specified index */
DEX_INLINE const DexClassDef* dexGetClassDef(const DexFile* pDexFile, u4 idx) {
    assert(idx < pDexFile->pHeader->classDefsSize);
    return &pDexFile->pClassDefs[idx];
}
```

而这里的pClassDefs是怎么来的呢？

```c++
/*
 * Set up the basic raw data pointers of a DexFile. This function isn't
 * meant for general use.
 */
void dexFileSetupBasicPointers(DexFile* pDexFile, const u1* data){
    DexHeader *pHeader = (DexHeader*) data;
    ... ...
    pDexFile->pClassDefs = (const DexClassDef*) (data + pHeader->classDefsOff);
    ...
}
```

由此可以看出，一个类的所有DexClassDef，也就是类定义，是从`pHeader->classDefsOff`偏移处开始，呈线性排列的，一个dex里面一共有`pHeader->ClassDefsSiz`个类定义。由此，我们就可以直接找到`pHeader->classDefsOff`偏移处，遍历所有的`DexClassDef`，如果发现这个`DexClassDef`的类名包含在补丁包中，就把它移除。

接下来，只要修改`pHeader->classDefsSiz`，把dex中类的数目改为去除补丁中的类之后的数目即可。

只去除类的定义，类的实体以及其他dex信息不做移除。