## Instant Run资源修复的核心代码

* com/android/tools/fd/runtime/MonkeyPatcher.java

```java
public static void monkeyPatchExistingResources(@Nullable Context context,
   @Nullable String externalResourceFile,@Nullable Collection<Activity> activities) {

    if (externalResourceFile == null) {
        return;
    }

    try {
        // %% Part 1. 创建一个新的AssetManager，并通过反射调用addAssetPath添加/sdcard上的新资源包.
        //         这样就构造出了一个带新资源的AssetManager
        AssetManager newAssetManager = AssetManager.class.getConstructor().newInstance();
        Method mAddAssetPath = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
        mAddAssetPath.setAccessible(true);
        // 通过反射调用addAssetPath方法加载外部的资源
        if (((Integer) mAddAssetPath.invoke(newAssetManager, externalResourceFile)) == 0) {
            throw new IllegalStateException("Could not create new AssetManager");
        }

        // Kitkat needs this method call, Lollipop doesn't. However, it doesn't seem to cause any harm
        // in L, so we do it unconditionally.
        Method mEnsureStringBlocks = AssetManager.class.getDeclaredMethod("ensureStringBlocks");
        mEnsureStringBlocks.setAccessible(true);
        mEnsureStringBlocks.invoke(newAssetManager);

        // %% Part 2. 反射得到Activity中AssetManager的引用处，全部换成刚才新构建的newAssetManager
        if (activities != null) {
            for (Activity activity : activities) {
                Resources resources = activity.getResources();

                try {
                    // 反射得到Resources的AssetManager类型的mAssets字段
                    Field mAssets = Resources.class.getDeclaredField("mAssets");
                    mAssets.setAccessible(true);
                    // 将mAssets字段的引用替换为新创建的AssetManager
                    mAssets.set(resources, newAssetManager);
                } catch (Throwable ignore) {
                    Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                    mResourcesImpl.setAccessible(true);
                    Object resourceImpl = mResourcesImpl.get(resources);
                    Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                    implAssets.setAccessible(true);
                    implAssets.set(resourceImpl, newAssetManager);
                }
                    ... ...

                pruneResourceCaches(resources);
            }
        }

        // %% Part 3. 得到Resources的弱引用集合，把他们的AssetManager成员替换成newAssetManager
        // Iterate over all known Resources objects
        Collection<WeakReference<Resources>> references;
        if (SDK_INT >= KITKAT) {
            // Find the singleton instance of ResourcesManager
            Class<?> resourcesManagerClass = Class.forName("android.app.ResourcesManager");
            Method mGetInstance = resourcesManagerClass.getDeclaredMethod("getInstance");
            mGetInstance.setAccessible(true);
            Object resourcesManager = mGetInstance.invoke(null);
            try {
                Field fMActiveResources = resourcesManagerClass.getDeclaredField("mActiveResources");
                fMActiveResources.setAccessible(true);
                @SuppressWarnings("unchecked")
                ArrayMap<?, WeakReference<Resources>> arrayMap =
                        (ArrayMap<?, WeakReference<Resources>>) fMActiveResources.get(resourcesManager);
                references = arrayMap.values();
            } catch (NoSuchFieldException ignore) {
                Field mResourceReferences = resourcesManagerClass.getDeclaredField("mResourceReferences");
                mResourceReferences.setAccessible(true);
                //noinspection unchecked
                references = (Collection<WeakReference<Resources>>) mResourceReferences.get(resourcesManager);
            }
        } else {
            Class<?> activityThread = Class.forName("android.app.ActivityThread");
            Field fMActiveResources = activityThread.getDeclaredField("mActiveResources");
            fMActiveResources.setAccessible(true);
            Object thread = getActivityThread(context, activityThread);
            @SuppressWarnings("unchecked")
            HashMap<?, WeakReference<Resources>> map =
                    (HashMap<?, WeakReference<Resources>>) fMActiveResources.get(thread);
            references = map.values();
        }
        // 遍历并得到弱引用集合中的Resources，将Resources的mAsseets字段引用替换成新的AssetManager
        for (WeakReference<Resources> wr : references) {
            Resources resources = wr.get();
            if (resources != null) {
                // Set the AssetManager of the Resources instance to our brand new one
                try {
                    Field mAssets = Resources.class.getDeclaredField("mAssets");
                    mAssets.setAccessible(true);
                    mAssets.set(resources, newAssetManager);
                } catch (Throwable ignore) {
                    Field mResourcesImpl = Resources.class.getDeclaredField("mResourcesImpl");
                    mResourcesImpl.setAccessible(true);
                    Object resourceImpl = mResourcesImpl.get(resources);
                    Field implAssets = resourceImpl.getClass().getDeclaredField("mAssets");
                    implAssets.setAccessible(true);
                    implAssets.set(resourceImpl, newAssetManager);
                }

                resources.updateConfiguration(resources.getConfiguration(), resources.getDisplayMetrics());
            }
        }
    } catch (Throwable e) {
        throw new IllegalStateException(e);
    }}
```

* 创建一个新的AssetManager
* 通过反射调用`addAssetPath`方法加载外部（SD卡）的资源。
* 遍历Activity列表，得到每个Activity的Resources
* 通过反射得到Resources的AssetManager类型的`mAssets`字段
* 改写`mAssets`字段的引用为新的AssetManager
* 将Resources.Theme的`mAssets`字段的引用替换为新创建的AssetManager。
* 根据SDK版本的不同，用不同的方式得到Resources的弱引用集合，再遍历这个弱引用集合，将弱引用集合中的Resources的`mAssets`字段引用都替换成新创建的AssetManager。