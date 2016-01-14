Android资源的热修复
-----------------------------------------------------------------------------------------

热修复中可能会涉及到资源文件的替换，有两个问题：

一、到底能不能替换？这里是将主APP中的资源替换成Patch apk中的资源，可以实现么？

二、怎么替换，会不会有资源id冲突的问题？

本章就来讨论这两个问题。

加载patch apk时，和加载插件类似，可以参考我之前的两篇文章：

[http://blog.csdn.net/dingjikerbo/article/details/47757511](http://blog.csdn.net/dingjikerbo/article/details/47757511)
[http://blog.csdn.net/dingjikerbo/article/details/47783411](http://blog.csdn.net/dingjikerbo/article/details/47783411)

如下

```
DexClassLoader dexClassLoader = createDexClassLoader(pluginId, packageId, packagePath);
AssetManager assetManager = createAssetManager(packagePath);
Resources resources = createResources(assetManager);

private AssetManager createAssetManager(String dexPath) {
    try {
        AssetManager assetManager = AssetManager.class.newInstance();
        Method addAssetPath = assetManager.getClass().getMethod(
                "addAssetPath", String.class);
        addAssetPath.invoke(assetManager, dexPath);
        return assetManager;
    } catch (Exception e) {
        e.printStackTrace();
        return null;
    }

}

private Resources createResources(AssetManager assetManager) {
    Resources superRes = mContext.getResources();
    Resources resources = new Resources(assetManager,
            superRes.getDisplayMetrics(), superRes.getConfiguration());
    return resources;
}
```

可见这里专门为Patch apk创建了独立的AssetManager，并将这个apk的路径添加到AssetManager的AssetPath路径集合中。

这里有两个问题：

1. 主APP和Patch加载资源用的是不同的AssetManager和Resources，如果主APP和Patch Apk中的资源id有冲突，那么加载资源时会不会串了？

2. 同一个AssetManager下可以添加多个资源包，假如这些资源包之间存在相同的资源id，加载资源时会不会串呢？

要研究这两个问题，我们需要先了解一下AssetManager的实现以及资源加载机制。

每个AssetManager下可以添加多个资源包，而且为了查找资源时更快，一定会先解析这些资源包，然后生成索引便于之后的资源加载。这个索引的key也就是资源的id应该包含资源的package id，资源type id以及资源的entry id。为保证key的唯一性，这三者不能同时冲突。除了资源type id通常是固定的之外，资源的package id 和entry id应该是打包apk的时候生成的，所以我们重点要确认一下打包apk时是如何生成资源的package id和entry id的。不过我们可以设想一下，假如两个资源包中的资源非常多，多到可以填满所有的资源entry id，那么就一定会有entry id的冲突，那么为保证key的唯一性的最后一道关卡就只剩下资源的package id了。然而实践中发现，编译时生成的R.java中所有的资源都是以0x7F开头，这个就是package id了，可见打包时默认的package id都是0x7F，除非修改aapt为每个包指定不同的package id，否则很有可能资源冲突。

当然如果采用的是不同的AssetManager去加载资源包里的资源就不会有这种问题了，因为索引表都不一样，即便key一样也没有关系。

接下来，我们将过一下AssetManager的代码，这里只是为了了解资源加载的大致流程，所以会略去一些细节，如果想更深入了解，可以参考如下几篇文章：
[Android应用程序资源的编译和打包过程分析](http://blog.csdn.net/luoshengyang/article/details/8744683)
[Android应用程序资源管理器（Asset Manager）的创建过程分析](http://blog.csdn.net/luoshengyang/article/details/8791064)
[Android应用程序资源的查找过程分析](http://blog.csdn.net/luoshengyang/article/details/8806798)

```
public AssetManager() {
    init();
}

private native final void init();
```

可见，直接调用了native的init函数进行初始化

```
static void android_content_AssetManager_init(JNIEnv* env, jobject clazz)
{
    AssetManager* am = new AssetManager();

    am->addDefaultAssets();

    env->SetIntField(clazz, gAssetManagerOffsets.mObject, (jint)am);
}
```

这个init主要做了三件事，首先在Native层New了一个AssetManager，然后向其中addDefaultAssets，最后将这个AssetManager的指针设置到Java中的AssetManager类的mObject。我们先来看看AssetManager 的构造函数。

```
class AssetManager : public AAssetManager {
public:
    ..........
    Vector<asset_path> mAssetPaths;
    mutable ResTable* mResources;
    ResTable_config* mConfig;

    CacheMode       mCacheMode;         // is the cache enabled?
    SortedVector<AssetDir::FileInfo> mCache;
    AssetManager(CacheMode cacheMode = CACHE_OFF);
    ..........
}
```

这个AssetManager中比较重要的是mAssetPaths和mResources。mAssetPaths中保存的是这个AssetManager维护的所有资源apk的路径。mResources就是一个Resource Table，相当于资源的索引表。再来看addDefaultAssets：

```
bool AssetManager::addDefaultAssets()
{
    const char* root = getenv("ANDROID_ROOT");

    String8 path(root);
    path.appendPath(kSystemAssets);

    return addAssetPath(path, NULL);
}

static const char* kSystemAssets = "framework/framework-res.apk";
```

可见这里所谓的defaultAssets就是${ANDROID_ROOT}/framework/framework-res.apk，是系统资源路径。

接下来重点讲解这个addAssetPath函数，在分析这个函数代码之前，我们先来看看那些地方调用了它：

一、addDefaultAssets时调用了addAssetPath(path, NULL)

二、Java层的AssetManager调用addAssetPath时调到了native层的android_content_AssetManager_addAssetPath如下

```
static jint android_content_AssetManager_addAssetPath(JNIEnv* env, jobject clazz,
                                                       jstring path)
{
    ScopedUtfChars path8(env, path);

    AssetManager* am = assetManagerForJavaObject(env, clazz);

    void* cookie;
    bool res = am->addAssetPath(String8(path8.c_str()), &cookie);

    return (res) ? (jint)cookie : 0;
}
```

这两个地方调用方式略有不同，前者第二个参数传NULL，后者添加用户自定义的path时传入了&cookie，是个void *，看来要返回一个指针，至于这个指针是做什么的，我们只能在看了addAssetPath代码后才能知道。

```
bool AssetManager::addAssetPath(const String8& path, void** cookie) {
    for (size_t i=0; i < mAssetPaths.size(); i++) {
        if (mAssetPaths[i].path == path) {
            if (cookie) {
                *cookie = (void*)(i+1);
            }
            return true;
        }
    }

    mAssetPaths.add(ap);

    if (cookie) {
        *cookie = (void*)mAssetPaths.size();
    }
    
    return true;
}
```

从代码中可以知道，如果当前要添加的path已经存在于AssetManager的mAssetPaths中了，就直接返回，否则将path添加到mAssetPaths末尾。cookie的作用就是标识这个path在mAssetPaths中的index。

好了，我们自定义资源apk的路径已经添加到AssetManager中了，那么我们是如何加载资源的呢？我们就拿Resources.getDrawable来作为入口。

```
public Drawable getDrawable(int id) {
    TypedValue value;
    synchronized (mAccessLock) {
        value = mTmpValue;
        if (value == null) {
            value = new TypedValue();
        } else {
            mTmpValue = null;
        }
        getValue(id, value, true);
    }
    final Drawable res = loadDrawable(value, id, theme);
    synchronized (mAccessLock) {
        if (mTmpValue == null) {
            mTmpValue = value;
        }
    }
    return res;
}
```

这里面getValue传入资源id，返回一个TypedValue，然后再拿着这个资源id和TypedValue去loadDrawable。我们先瞧瞧getValue是干什么的？

```
public void getValue(int id, TypedValue outValue, boolean resolveRefs) {
    boolean found = mAssets.getResourceValue(id, 0, outValue, resolveRefs);
    if (found) {
        return;
    }
    throw new NotFoundException("Resource ID #0x"
                                + Integer.toHexString(id));
}
```

这里面调用了AssetManager的getResourceValue，如下：

```
boolean getResourceValue(int ident, int density, TypedValue outValue, boolean resolveRefs)
{
    loadResourceValue(ident, (short) density, outValue, resolveRefs);
	.............
}
```

看来loadResourceValue里会设置这个outValue：

```
private native final int loadResourceValue(int ident, short density, TypedValue outValue,
            boolean resolve);
```

这个函数是个native的，实现如下：

```
static jint android_content_AssetManager_loadResourceValue(JNIEnv* env, jobject clazz,
                                                           jint ident,
                                                           jshort density,
                                                           jobject outValue,
                                                           jboolean resolve)
{
    AssetManager* am = assetManagerForJavaObject(env, clazz);
    const ResTable& res(am->getResources());

    Res_value value;
    ResTable_config config;
    uint32_t typeSpecFlags;
    ssize_t block = res.getResource(ident, &value, false, density, &typeSpecFlags, &config);
    copyValue(env, outValue, &res, value, ident, block, typeSpecFlags, &config);
}
```

看起来貌似有点复杂，调了不少函数，不过没关系，我们耐心点分析。这个函数主要做了四件事：
1. 通过assetManagerForJavaObject拿到AssetManager对象。
2. 通过am->getResources()获取AssetManager中的ResTable。
3. 调用ResTable的getResource函数获取资源相关的索引和配置信息。
4. 调用copyValue将一些资源属性拷贝到TypedValue中。

先来看看am->getResources()的实现如下：

```
const ResTable& AssetManager::getResources(bool required) const
{
    const ResTable* rt = getResTable(required);
    return *rt;
}

const ResTable* AssetManager::getResTable(bool required) const
{
    ResTable* rt = mResources;
    if (rt) {
        return rt;
    }

    if (mResources != NULL) {
        return mResources;
    }

    const size_t N = mAssetPaths.size();

    for (size_t i=0; i < N; i++) {
        Asset* ass = NULL;
        ResTable* sharedRes = NULL;
        
        ..........

        if ((ass != NULL || sharedRes != NULL) && ass != kExcludedAsset) {
            if (rt == NULL) {
                mResources = rt = new ResTable();
            }
            if (sharedRes != NULL) {
                rt->add(sharedRes);
            } else {
                rt->add(ass, (void*)(i+1), !shared, idmap);
            }
        }
    }

    if (!rt) {
        mResources = rt = new ResTable();
    }
    return rt;
}
```

这里省略了不少代码，不过不影响我们掌握大致流程。首先判断资源索引表是否已经生成了，如果已生成则直接返回，否则要生成索引表。遍历mAssetPaths，依次解析资源文件，将生成的数据结构和索引合并到总的mResources中。

好了，Resource Table准备好了，接下来看看如何从Resource Table中getResource

```
ssize_t ResTable::getResource(uint32_t resID, Res_value* outValue, bool mayBeBag, uint16_t density,
        uint32_t* outSpecFlags, ResTable_config* outConfig) const
{
    const ssize_t p = getResourcePackageIndex(resID);
    const int t = Res_GETTYPE(resID);
    const int e = Res_GETENTRY(resID);

    const Res_value* bestValue = NULL;
    const Package* bestPackage = NULL;
    ResTable_config bestItem;

    const PackageGroup* const grp = mPackageGroups[p];
    const ResTable_config* desiredConfig = &mParams;
    ResTable_config* overrideConfig = NULL;
    if (density > 0) {
        overrideConfig = (ResTable_config*) malloc(sizeof(ResTable_config));
        memcpy(overrideConfig, &mParams, sizeof(ResTable_config));
        overrideConfig->density = density;
        desiredConfig = overrideConfig;
    }

    ssize_t rc = BAD_VALUE;
    size_t ip = grp->packages.size();
    while (ip > 0) {
        ip--;
        int T = t;
        int E = e;

        const Package* const package = grp->packages[ip];
        const ResTable_type* type;
        const ResTable_entry* entry;
        const Type* typeClass;
        ssize_t offset = getEntry(package, T, E, desiredConfig, &type, &entry, &typeClass);

		..........
        
        bestItem = thisConfig;
        bestValue = item;
        bestPackage = package;
    }

    if (bestValue) {
        outValue->size = dtohs(bestValue->size);
        outValue->res0 = bestValue->res0;
        outValue->dataType = bestValue->dataType;
        outValue->data = dtohl(bestValue->data);
        if (outConfig != NULL) {
            *outConfig = bestItem;
        }
        rc = bestPackage->header->index;
        goto out;
    }

out:
    if (overrideConfig != NULL) {
        free(overrideConfig);
    }

    return rc;
}
```

这个函数有点长，不过不难看出主要做了这么几件事，首先通过资源ID获取到资源的package id、类型id、entry id。然后通过package id取到对应的PackageGroup，接着遍历PackageGroup下的所有Packages查找资源，找出最匹配的那个资源项并返回。至于是怎么查找最匹配资源项的这里不是我们关注的重点，就不赘述了。

最后再来看看这个copyValue，主要是设置这个Java中传下来的outValue对象，这个outValue对应Java中的TypedValue。

```
jint copyValue(JNIEnv* env, jobject outValue, const ResTable* table,
               const Res_value& value, uint32_t ref, ssize_t block,
               uint32_t typeSpecFlags, ResTable_config* config)
{
    env->SetIntField(outValue, gTypedValueOffsets.mType, value.dataType);
    env->SetIntField(outValue, gTypedValueOffsets.mAssetCookie,
                        (jint)table->getTableCookie(block));
    env->SetIntField(outValue, gTypedValueOffsets.mData, value.data);
    env->SetObjectField(outValue, gTypedValueOffsets.mString, NULL);
    env->SetIntField(outValue, gTypedValueOffsets.mResourceId, ref);
    env->SetIntField(outValue, gTypedValueOffsets.mChangingConfigurations,
            typeSpecFlags);
    if (config != NULL) {
        env->SetIntField(outValue, gTypedValueOffsets.mDensity, config->density);
    }
    return block;
}
```

最后，我们来看看Resources的loadDrawable是如何实现的：

```
Drawable loadDrawable(TypedValue value, int id)
        throws NotFoundException {

    boolean isColorDrawable = false;
    if (value.type >= TypedValue.TYPE_FIRST_COLOR_INT &&
            value.type <= TypedValue.TYPE_LAST_COLOR_INT) {
        isColorDrawable = true;
    }
    final long key = isColorDrawable ? value.data :
            (((long) value.assetCookie) << 32) | value.data;

    Drawable dr = getCachedDrawable(isColorDrawable ? mColorDrawableCache : mDrawableCache, key);

    if (dr != null) {
        return dr;
    }
    
    Drawable.ConstantState cs;
    if (isColorDrawable) {
        cs = sPreloadedColorDrawables.get(key);
    } else {
        cs = sPreloadedDrawables[mConfiguration.getLayoutDirection()].get(key);
    }
    if (cs != null) {
        dr = cs.newDrawable(this);
    } else {
        if (isColorDrawable) {
            dr = new ColorDrawable(value.data);
        }

        if (dr == null) {
            String file = value.string.toString();

            if (file.endsWith(".xml")) {
                XmlResourceParser rp = loadXmlResourceParser(
                        file, id, value.assetCookie, "drawable");
                dr = Drawable.createFromXml(this, rp);
                rp.close();
            } else {
                InputStream is = mAssets.openNonAsset(
                        value.assetCookie, file, AssetManager.ACCESS_STREAMING);
                dr = Drawable.createFromResourceStream(this, value, is,
                        file, null);
                is.close();
            }
        }
    }

    .........

    return dr;
}
```

这个函数首先从缓存中查找资源，如果找到了就返回，找不到的话就去打开资源文件读取资源，读出来后放到缓存中，逻辑很简单，只不过不同的资源处理方式不同而已。

总结一下，整个资源加载的流程：
一、获取资源时首先看缓存中有不有，如果有就直接返回，没有就去加载资源
二、加载资源时首先看资源的索引表是否建立了，如果建立了就直接根据索引和配置信息去加载资源，否则就解析并建立资源索引

要注意的是：
一、资源索引的建立是以AssetManager为单位的，同一个AssetManager内如果有多个资源包，则要保证这些包之间不会有id冲突，打包时需要考虑怎么给每个包设置不同的package id。如果不想这么做，就给每个包分别赋予一个AssetManager，就不用担心id冲突的问题。

接下来，我会给出一个Demo，用于展示如何进行资源的热修复。

先建立主App工程，如下：

```
public class MainActivity extends Activity {

    private Button mBtnPatch;
    private Button mBtnShow;

    private TextView mTvText;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mBtnPatch = (Button) findViewById(R.id.patch);
        mBtnShow = (Button) findViewById(R.id.show);
        mTvText = (TextView) findViewById(R.id.text);

        mBtnPatch.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                patch();
            }
        });

        mBtnShow.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                showName();
            }
        });
    }

    private void patch() {
        File dir = getExternalCacheDir();

        if (!dir.exists() && !dir.mkdirs()) {
            return;
        }

        File patch = new File(getExternalCacheDir(), "Patch1.apk");

        if (patch.exists() && patch.isFile()) {
            PatchMain.load(this, patch.getAbsolutePath(), null);
        }
    }

    public void showName() {
        mTvText.setText(R.string.host);
    }
}
```

这里添加了两个按钮，一个按钮触发Patch，一个按钮触发调用某个函数，这个函数是我们之后要Hook的。

再来建立Patch工程，如下：

```
public class Patch1 extends Patch {

	@Override
	public void handlePatch(PatchParam arg0) throws Throwable {
		// TODO Auto-generated method stub
		final Class<?> clazz = arg0.context.getClassLoader().loadClass(
				"com.example.hookresearch.MainActivity");
		
		final Context context = arg0.context;
		
		DexposedBridge.findAndHookMethod(clazz,  "showName", new XC_MethodReplacement() {

			@Override
			protected Object replaceHookedMethod(MethodHookParam arg0)
					throws Throwable {
				// TODO Auto-generated method stub
				Field field = FieldUtils.getDeclaredField(clazz, "mTvText", true);
				TextView textView = (TextView) field.get(context);
				
				String text = getPatchString(R.string.patch_world);
				Log.i("bush", "patch string is " + text);
				textView.setText(text);
				
				Log.i("bush", "host string is " + getHostString("host"));
				
				return null;
			}
			
		});
	}
}
```

这个Patch apk中暂时只有一个Patch补丁类，是继承自Patch类的。主要是为了修复MainActivity中mTvText显示的字符串，可以让其显示主App中的另一个字符串，也可以让其显示Patch apk中的字符串。这里首先要获取目标TextView，需要通过反射实现，这个反射用的是apache common lang的开源库。获取TextView后，接下来就要获取String资源了，Patch类提供了一些接口用于加载资源，包括加载Host apk中的资源和Patch apk中的资源。区别在于加载Patch apk中的资源可以直接指定资源id，但是加载Host apk中的资源需要指定资源的name。

接下来，我们看看这个Patch框架的实现，在Host apk中加载补丁时调用的是PatchMain.load函数，如下：

```
public static PatchResult load(Context context, String apkPath, HashMap<String, Object> contentMap) {
    PatchResult result = loadAllCallbacks(context, apkPath, context.getClassLoader());
    PatchParam lpparam = new PatchParam(loadedPatchCallbacks);
    lpparam.context = context;
    lpparam.contentMap = contentMap;
    return PatchCallback.callAll(lpparam);
}
```

看起来很简单，就是解析补丁apk，生成PatchResult，然后传入参数，调用所有的补丁。接下来看看这个loadAllCallbacks是如何实现的：

```
private static PatchResult loadAllCallbacks(Context context, String apkPath, ClassLoader cl) {
        try {
            File e = new File(apkPath + "odex");
            if(e.exists()) {
                e.delete();
            }

            DexClassLoader mcl = null;
            PatchContext patchContext = null;

            try {
                mcl = new DexClassLoader(apkPath, context.getFilesDir().getAbsolutePath(), (String)null, cl);
                AssetManager assetManager = createAssetManager(apkPath);
                Resources resources = createResources(context, assetManager);
                patchContext = new PatchContext(mcl, assetManager, resources);
            } catch (Throwable var11) {
                return new PatchResult(false, PatchResult.FOUND_PATCH_CLASS_EXCEPTION, "Find patch class exception ", var11);
            }

            DexFile dexFile = DexFile.loadDex(apkPath, apkPath + "odex", 0);
            Enumeration entrys = dexFile.entries();
            ReadWriteSet entry = loadedPatchCallbacks;
            synchronized(loadedPatchCallbacks) {
                loadedPatchCallbacks.clear();
            }

            while(entrys.hasMoreElements()) {
                String entry1 = (String)entrys.nextElement();
                Class entryClass = null;

                try {
                    entryClass = mcl.loadClass(entry1);
                } catch (ClassNotFoundException var12) {
                    var12.printStackTrace();
                    break;
                }

                if (Patch.class.isAssignableFrom(entryClass)) {
                    Patch module = (Patch) entryClass.newInstance();
                    module.context = context;
                    module.patch = patchContext;
                    hookLoadPatch(new PatchCallback(module));
                }
            }
        } catch (Exception var13) {
            return new PatchResult(false, PatchResult.FOUND_PATCH_CLASS_EXCEPTION, "Find patch class exception ", var13);
        }

        return new PatchResult(true, PatchResult.NO_ERROR, "");
    }

private static void hookLoadPatch(PatchCallback callback) {
    ReadWriteSet var1 = loadedPatchCallbacks;
    synchronized(loadedPatchCallbacks) {
        loadedPatchCallbacks.add(callback);
    }
}
```

这个代码稍微麻烦点，不过主要做了两件事：
一、用DexClassLoader加载patch apk，生成对应的Resources便于之后加载patch apk中的资源
二、解析出patch apk中所有的补丁类，补丁类都是继承自Patch类。所有补丁类都会加到一个集合中。

接下来，就是遍历所有的补丁类，依次调用他们的handlePatch接口来打补丁。

最后，我们来看看Patch的实现：

```
public abstract class Patch implements IPatch {

    public Context context;

    public PatchContext patch;

    public String getHostString(String name) {
        Resources resources = getHostResources();
        if (resources != null) {
            int resId = resources.getIdentifier(name, "string", getHostPackageName());
            if (resId > 0) {
                return resources.getString(resId);
            }
        }
        return "";
    }

    public Drawable getHostDrawable(String name) {
        Resources resources = getHostResources();
        if (resources != null) {
            int resId = resources.getIdentifier(name, "drawable", getHostPackageName());
            if (resId > 0) {
                return resources.getDrawable(resId);
            }
        }
        return null;
    }

    public String getPatchString(int resId) {
        Resources resources = getPatchResources();
        if (resources != null && resId > 0) {
            return resources.getString(resId);
        }
        return "";
    }

    public Drawable getPatchDrawable(int resId) {
        Resources resources = getPatchResources();
        if (resources != null && resId > 0) {
            return resources.getDrawable(resId);
        }
        return null;
    }

    public Resources getHostResources() {
        return context != null ? context.getResources() : null;
    }

    public Resources getPatchResources() {
        return patch != null ? patch.resources : null;
    }

    public String getHostPackageName() {
        return context != null ? context.getPackageName() : "";
    }

}
```












