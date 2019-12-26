## 第4章 资源热修复技术

### 1 普遍的实现方式

Instant Run中的资源热修复分为两步：

* 1.构造一个新的AssetManager，并通过反射调用addAssetPath，把这个完整的新资源包加入到AssetManager中。这样就得到一个含有所有新资源的AssetManager。
* 找到所有之前应用到原有AssetManager的地方，通过反射，把引用处替换为新AssetManager。

大量的代码都是在处理兼容性问题，并找到所有AssetManager的引用处。其中的重点，是addAssetPath这个函数，底层实现逻辑如下：

以Android6.0为例，addAssetPath最终调用到了native方法。

文件：frameworks/base/core/java/android/content/res/AssetManager.java。

```java
    /**
     * Add an additional set of assets to the asset manager.  This can be
     * either a directory or ZIP file.  Not for use by applications.  Returns
     * the cookie of the added asset, or 0 on failure.
     * {@hide}
     */
    public final int addAssetPath(String path) {
        synchronized (this) {
            int res = addAssetPathNative(path);
            makeStringBlocks(mStringBlocks);
            return res;
        }
    }
    private native final int addAssetPathNative(String path);
```

执行addAssetPath就是解析这个格式，然后高早出底层数据结构的过程。整个解析资源调用链是：

* public final int addAsset(String path)
* android_content_AssetManager_addAssetPath
* AssetManager::addAssetPath
* AssetManager::AppendPathToResTable
* ResTable::add
* ResTable::addInternal
* RetTable::parsePackage

大致过程就是，通过传入的资源包路径，先得到其中的`resources.arsc`，然后解析它的格式，存放在底层的AssetManager的mResources成员中。

```c++
@frameworks/base/include/androidfw/AssetManager.h
class AssetManager : public AAssetManager {
    ... ...
    mutable ResTable* mResources;
    ... ...
}
```

AssetManager的mResources成员是一个ResTable结构体：

```c++
@frameworks/base/include/androidfw/ResourceTypes.h
class ResTable {
    mutable Mutex               mLock;
    // 互斥锁，用于多进程间互斥操作
    
    status_t                    mError;

    ResTable_config             mParams;

    // Array of all resource tables.
    Vector<Header*>             mHeaders;
    /* 表示所有resources.arsc原始数据，这就等同于所有通过addAssetPath加载进来的路径的资源ID信息 */
    
    // Array of packages in all resource tables.
    Vector<PackageGroup*>       mPackageGroups;
    // 资源包的实体，包含所有加载进来的package id所对应的资源 
    
    // Mapping from resource package IDs to indices into the internal
    // package array.
    uint8_t                     mPackageMap[256];
    /* 索引表，表示0~256的package id,每个元组分别存放该ID所属PackageGroup在
       mPackageGroups中的index */
    
    uint8_t                     mNextPackageId;
};
```

**一个Android进程只包含一个ResTable**，ResTable的成员变量mPackageGroups就是所有解析过的资源包的集合。任何一个资源包中都含有resources.arsc，它记录了所有资源的ID分配情况，以及资源中的所有字符串。这些信息是以二进制数的方式存储。底层的AssetManager做的事就是解析这个文件，然后把相关信息存储到mPackageGroups里面。

***

### 2 资源文件的格式

整个`resources.arsc`文件，是由一个个ResChunk拼接起来的。从文件头开始，每个chunk的头部都是一个ResChunk_header结构，它只是了这个chunk的大小和数据类型。

```c++
@frameworks/base/include/androidfw/ResourceTypes.h
/**
 * Header that appears at the front of every data chunk in a resource.
 */
struct ResChunk_header
{
    // Type identifier for this chunk.  The meaning of this value depends
    // on the containing chunk.
    uint16_t type;

    // Size of the chunk header (in bytes).  Adding this value to
    // the address of the chunk allows you to find its associated data
    // (if any).
    uint16_t headerSize;

    // Total size of this chunk (in bytes).  This is the chunkSize plus
    // the size of any data associated with the chunk.  Adding this value
    // to the chunk allows you to completely skip its contents (including
    // any child chunks).  If this value is the same as chunkSize, there is
    // no data associated with the chunk.
    uint32_t size;
};
```

通过Reschunk_header中的type成员，可以知道这个chunk是什么类型，从而就知道应该如何解析这个chunk。

解析完一个chunk，从这个chunk+size的位置开始，就是下一个chunk起始位置。

一个resources.arsc里面包含若干个package，不过默认情况下，由打包工具AAPT打出来的包只有一个package。这个package包含了App中所有资源信息。

资源信息主要是指每个资源的名称以及它对应的编号。编号是一个32位数组，用十六进制来表示就是0xPPTTEEEE。`PP`为`package id`，`TT`为`type id`，`EEEE`为`entry id`。

* **package id**，每个package对应的是类型为RES_TABLE_PACKAGE_TYPE的ResTable_package结构体，ResTable_package结构体的ID成员变量就表示它的package id。
* **type id**，每个type对应的是类型为RES_TABLE_TYPE_SPEC_TYPE的ResTable_package结构体，它的ID成员变量就是type id。但是，该type id具体对应什么类型，是需要到package chunk里的Type String Pool中去解析得到的。比如Type String pool中依次有attr、drawable、mipmap、layout字符串，就表示attr类型的type id为1，drawable类型的type id为2，mipmap类型的type id为3，layout类型的type id为4。所以，每个type id对应了Type String Pool里的字符顺序所指定的类型。
* **entry id**，每个entry表示一个资源项，资源项是按照排列的先后顺序自动被标记编号的。也就是说，一个type里按位置出现的第一个资源项，其entry id为0x0000，第二个为0x0001，依次类推。因此我们是无法直接指定entry id的，只能够根据排布顺序决定。资源项之间是紧密排布的，没有空隙，但是可以指定资源项为ResTable_type::NO_ENTRY来填入一个空资源。

***

### 3 运行时资源的解析

默认由Android SDK编出来的APK，是由AAPT工具进行打包的，起资源包的package id就是**0x7f**。

系统资源包，也就是frameworks-res.jar，package id为**0x01**。

在走到App第一行代码之前，系统就已经帮我们构建好一个已经添加了安装包资源的AssetManager了。

```java
@frameworks/base/core/java/android/app/ResourcesManager.java
    Resources getTopLevelResources(String resDir, String[] splitResDirs,
            String[] overlayDirs, String[] libDirs, int displayId,
            Configuration overrideConfiguration, CompatibilityInfo compatInfo) {
        AssetManager assets = new AssetManager();
        // resDir就是安装包APK
        if (resDir != null) {
            if (assets.addAssetPath(resDir) == 0) {
                return null;
            }
        }
```

如果直接在原有的AssetManager上接续addAssetPath的完整补丁包的话，由于补丁包的package id也是0x7f，所以使得同一个package id的包被加载两次。

* 在Android L之后，它会默默的把后来的包添加到之前包的同一个PackageGroup下面。在解析的时候，会与之前的包比较同一个type id所对应的类型，如果该类型下的资源数目和之前添加过的不一致，会打出一条warning log，但是仍旧加入到该类行的TypeList中。

  在获取某个Type的资源时，会从前往后遍历，也就是说先得到原有安装包里的资源，除非后面的资源的config比前面的更详细才会发生覆盖。而对于同一个config而言，补丁中的资源就永远无法生效了。所以在Android L以上的版本，在原有AssetManager上加入补丁包，是无效的。

* 在Android 4.4及以下版本，addAssetPath只是把补丁包的路径添加到了mAssetPath中，而真正解析的资源包的逻辑是在App第一次执行`AssetManager::getResTable`的时候。

  而在执行到加载补丁代码的时候，getResTable已经执行过了无数次。这是因为就算我们之前没有做过任何资源相关操作，Android framework里的代码也会多次调用到哪里。所以，以后即使是addAssetPath，也只是添加到了mAssetPath，并不会发生解析。因而补丁包里面的资源是完全不生效的。

**像Instant Run这种方案，一定需要一个全新的AssetManager时，再加入完整的新资源包，替换掉原有的AssetManager**。

***

### 4 另辟蹊径的资源修复方案

一个好的资源热修复方案是什么样？

* 补丁包足够小（避免下发完整的补丁包）
* 避免运行时占用很多资源（避免下发差量包运行时合成）
* 不侵入打包流程（避免自行修改AAPT）

***新方案***

> 构建一个package id为0x66的资源包，这个包只包含改变了的资源项，直接在原有AssetManager中addAssetPath这个包即可。

* 对于新增资源，直接加入补丁包，然后新代码里直接引用就可以了，没什么好说的。
* 对于减少资源，不使用它就可以了。
* 对于修改资源，比如替换一张图片之类的情况。我们把它视为新增资源，在打入补丁的时候，代码在引用处也会做相应修改，也就是直接把原来使用旧资源ID的地方变为新ID。

![图](https://raw.githubusercontent.com/NieJianJian/AndroidNotes/master/Picture/sophix_patchpackage.png)

补丁包示意图如上，绿线表示新增资源，红线表示内容发生修改的资源，黑线表示内容没有变化，但是ID发生改变的资源，× 表示删除了资源。

### 4.1 新增资源及其导致的ID偏移

　　新资源的插入的位置是随机的，这与每次AAPT打包时解析XML的顺序有关。所以会导致一些原有的资源ID发生偏移，如上图中，新增`holo_grey`导致`holo_light`的ID发生偏移，原本代码中：

```java
imageView.setImageResource(R.drawable.holo_light);
```

等价于：

```java
imageView.setImageResource(0x7f020002);
```

由于新资源`holo_grey`的插入，导致引用变成了：

```java
imageView.setImageResource(0x7f020003);
```

但是这种情况不属于资源改变，也不属于代码改变，所以我们在对比新旧代码之前，会把新包里的这行代码修正会原来的ID：

```java
imageView.setImageResource(0x7f020002);
```

然后进行后续代码对比，这样后续代码对比时旧不会检测到发生了改变。

### 4.2 内容发生改变的资源

对于内容发生改变的资源（类型为layout的`activity_main`的文件内容，或者类型为String的`no`的值），它们都会加入补丁包，并重新编号为新ID。相应代码也会发生改变，比如：

```java
setContentView(R.layout.activity_main);
```

等价于：

```java
setContentView(0x7f030000);
```

在生成对比新旧代码之前，我们会把新包里面的这行代码变为：

```java
setContentView(0x66020000);
```

这样就可以引用到新内容资源了。

### 4.3 删除了的资源

不用管他，不使用它就可以了。

### 4.4 对于type的影响

