关于Android PreloadedDrawables
---------------------------------------------

研究preloadedDrawables是因为要在单独进程中启动一个service，而这个service的内存占用有严格限制，经过研究发现内存占用中很大一部分是preloadedResources，而service是后台运行无需界面的，所以这块内存可以清理掉。系统没有提供现成的接口，所以需要通过反射实现。

首先，我们做一个实验来验证效果：

```
public class MainActivity extends Activity {

	private Button mBtnStart;
	private Button mBtnStop;
	private Button mBtnClear;

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);

		mBtnStart = (Button) findViewById(R.id.start);
		mBtnStop = (Button) findViewById(R.id.stop);
		mBtnClear = (Button) findViewById(R.id.clear);

		mBtnStart.setOnClickListener(new View.OnClickListener() {

			@Override
			public void onClick(View v) {
				// TODO Auto-generated method stub
				Intent startIntent = new Intent(MainActivity.this,
						MyService.class);
				startService(startIntent);
			}
		});

		mBtnStop.setOnClickListener(new View.OnClickListener() {

			@Override
			public void onClick(View v) {
				// TODO Auto-generated method stub
				Intent stopIntent = new Intent(MainActivity.this,
						MyService.class);
				stopService(stopIntent);
			}
		});

		mBtnClear.setOnClickListener(new View.OnClickListener() {

			@Override
			public void onClick(View v) {
				// TODO Auto-generated method stub
				sendBroadcast();
			}
		});
	}
	
	private void sendBroadcast() {
		Intent intent = new Intent("com.example.clear");
		this.sendBroadcast(intent);
	}
}
```

这里有三个按钮，分别是启动service，停止service，和清理service的preloaded resource。再来看看service的实现：

```
public class MyService extends Service {
	
	@Override
	public void onCreate() {
		// TODO Auto-generated method stub
		super.onCreate();
		Log.i("bush", "service onCreate");
		
		IntentFilter filter = new IntentFilter("com.example.clear");
		registerReceiver(mReceiver, filter);
	}
	
	private final BroadcastReceiver mReceiver = new BroadcastReceiver() {

		@Override
		public void onReceive(Context context, Intent intent) {
			// TODO Auto-generated method stub
			PreloadClearUtil.clearResources();
			Log.i("bush", "service onReceive");
		}
	};
	
	@Override
	public void onDestroy() {
		// TODO Auto-generated method stub
		super.onDestroy();
		Log.i("bush", "service onDestroy");
		
		unregisterReceiver(mReceiver);
	}

	@Override
	public IBinder onBind(Intent intent) {
		// TODO Auto-generated method stub
		return null;
	}

}
```

这里面什么也没有做，只是接收清理资源的广播而已，service运行在单独的进程中。我们启动app，点击start service按钮，发现DDMS中多出来一个进程，如下：

![这里写图片描述](http://img.blog.csdn.net/20160119105708697)

我们选中service所在的进程，Update Heap，并GC一下，查看内存占用情况

![这里写图片描述](http://img.blog.csdn.net/20160119105818322)

这里可见我们的service里什么也没有，内存却占用了约23M。我们Dump Hprof File，如下：

![这里写图片描述](http://img.blog.csdn.net/20160119110139245)

可见这些内存主要是sPreloadedDrawables持有，而且都是强引用。

我们点击按钮清理这些preloadedDrawables，再查看内存占用情况，如下：

![这里写图片描述](http://img.blog.csdn.net/20160119110340853)

![这里写图片描述](http://img.blog.csdn.net/20160119110430260)

可见内存只占用3M左右，大部分都是必须加载的Class和String资源了。

接下来，我们再看看这些preloadedDrawables是什么时候加载的。

Zygote启动时会调用ZygoteInit的main函数中，其中会通过preloadResources()来预加载资源，而之后所有的android应用进程都是从Zygote fork出来的，所以就继承了这部分预加载的资源，包括独立的Service进程。

```
private static void preloadResources() {
    final VMRuntime runtime = VMRuntime.getRuntime();
    mResources = Resources.getSystem();
    mResources.startPreloading();
    if (PRELOAD_RESOURCES) {
        TypedArray ar = mResources.obtainTypedArray(
                com.android.internal.R.array.preloaded_drawables);
        int N = preloadDrawables(runtime, ar);
        ar = mResources.obtainTypedArray(
                com.android.internal.R.array.preloaded_color_state_lists);
        N = preloadColorStateLists(runtime, ar);
    }
    mResources.finishPreloading();
}

private static final boolean PRELOAD_RESOURCES = true;
```

这里首先获取system resources，然后依次预加载Drawables和ColorStateLists。是否预加载的开关PRELOAD_RESOURCES默认是打开的。
预加载的资源在com.android.internal.R.array.preloaded_drawables中声明。

```
private static Resources mSystem = null;

public static Resources getSystem() {
    synchronized (mSync) {
        Resources ret = mSystem;
        if (ret == null) {
            ret = new Resources();
            mSystem = ret;
        }

        return ret;
    }
}
```

Resources中有一个静态全局变量mSystem，表示系统的资源。接下来我们看看preloadDrawables的实现，如下：

```
private static int preloadDrawables(VMRuntime runtime, TypedArray ar) {
    int N = ar.length();
    for (int i=0; i<N; i++) {
        int id = ar.getResourceId(i, 0);
        if (id != 0) {
            Drawable dr = mResources.getDrawable(id);
        }
    }
    return N;
}
```

这里依次获取预加载的资源id，然后调用mResources.getDrawable(id)加载资源，资源加载后就会保存在Resources的缓存中。如下：

```
public Drawable getDrawable(int id) throws NotFoundException {
    synchronized (mTmpValue) {
        TypedValue value = mTmpValue;
        getValue(id, value, true);
        return loadDrawable(value, id);
    }
}
```

这个函数的解析我在这篇文章中讲过：[Xposed研究——如何进行资源的热修复](http://blog.csdn.net/dingjikerbo/article/details/50498775)

在loadDrawable中，首先看mDrawableCache中有没有，有的话直接返回，没有的话再去看sPreloadedDrawables缓存中有没有，如果有就返回，没有的话再去加载资源。值得注意的是加载资源时要区分是否是预加载的，预加载的资源是通过强引用放在sPreloadedDrawables中的，用户自定义的资源是通过弱引用放在mDrawableCache中的。如下：

```
Drawable loadDrawable(TypedValue value, int id)
            throws NotFoundException {
    final long key = (((long) value.assetCookie) << 32) | value.data;
    Drawable dr = getCachedDrawable(key);

    if (dr != null) {
        return dr;
    }

    Drawable.ConstantState cs = sPreloadedDrawables.get(key);
    if (cs != null) {
        dr = cs.newDrawable(this);
    } else {
        if (value.type >= TypedValue.TYPE_FIRST_COLOR_INT &&
                value.type <= TypedValue.TYPE_LAST_COLOR_INT) {
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

    if (dr != null) {
        dr.setChangingConfigurations(value.changingConfigurations);
        cs = dr.getConstantState();
        if (cs != null) {
            if (mPreloading) {
                sPreloadedDrawables.put(key, cs);
            } else {
                synchronized (mTmpValue) {
                    mDrawableCache.put(key, new WeakReference<Drawable.ConstantState>(cs));
                }
            }
        }
    }

    return dr;
}

private static final LongSparseArray<Drawable.ConstantState> sPreloadedDrawables
        = new LongSparseArray<Drawable.ConstantState>();
        
private final LongSparseArray<WeakReference<Drawable.ConstantState> > mDrawableCache
        = new LongSparseArray<WeakReference<Drawable.ConstantState> >();

```

可见系统预加载的资源都是强引用缓存，而用户自定义的资源都是弱引用缓存。

这里缓存中保存的都是Drawable.ConstantState，不是Drawable，真正获取Drawable是通过ConstantState的newDrawable来完成的。

这里会升起一个疑问，预加载的难道只是一个外壳，而并没有分配内存，真正需要资源的时候才去分配内存么？我们来看看BitmapDrawable的实现吧：

```
final static class BitmapState extends ConstantState {
    Bitmap mBitmap;
    int mChangingConfigurations;
    int mGravity = Gravity.FILL;
    Paint mPaint = new Paint(DEFAULT_PAINT_FLAGS);
    Shader.TileMode mTileModeX = null;
    Shader.TileMode mTileModeY = null;
    int mTargetDensity = DisplayMetrics.DENSITY_DEFAULT;
    boolean mRebuildShader;
    boolean mAutoMirrored;

    BitmapState(Bitmap bitmap) {
        mBitmap = bitmap;
    }

    BitmapState(BitmapState bitmapState) {
        this(bitmapState.mBitmap);
        mChangingConfigurations = bitmapState.mChangingConfigurations;
        mGravity = bitmapState.mGravity;
        mTileModeX = bitmapState.mTileModeX;
        mTileModeY = bitmapState.mTileModeY;
        mTargetDensity = bitmapState.mTargetDensity;
        mPaint = new Paint(bitmapState.mPaint);
        mRebuildShader = bitmapState.mRebuildShader;
        mAutoMirrored = bitmapState.mAutoMirrored;
    }

    @Override
    public Bitmap getBitmap() {
        return mBitmap;
    }

    @Override
    public Drawable newDrawable() {
        return new BitmapDrawable(this, null);
    }

    @Override
    public Drawable newDrawable(Resources res) {
        return new BitmapDrawable(this, res);
    }

    @Override
    public int getChangingConfigurations() {
        return mChangingConfigurations;
    }
}
```

这里newDrawable是调用的BitmapDrawable的构造函数，如下：

```
private BitmapDrawable(BitmapState state, Resources res) {
    mBitmapState = state;
    if (res != null) {
        mTargetDensity = res.getDisplayMetrics().densityDpi;
    } else {
        mTargetDensity = state.mTargetDensity;
    }
    setBitmap(state != null ? state.mBitmap : null);
}
```

这里首先将这个BitmapState设置到BitmapDrawable中，然后调用BitmapDrawable的setBitmap，如下：

```
private void setBitmap(Bitmap bitmap) {
    if (bitmap != mBitmap) {
        mBitmap = bitmap;
        if (bitmap != null) {
            computeBitmapSize();
        } else {
            mBitmapWidth = mBitmapHeight = -1;
        }
        invalidateSelf();
    }
}
```

可见，ConstantState的newDrawable并没有分配内存去new Drawable，只是套了一层Drawable的壳，那么分配内存是在哪里做的呢？
上面loadDrawable时，如果缓存中没有，就会去资源文件中加载资源，想必是在这里面做的。这里分两种情况，当资源文件是xml时，会调用
Drawable.createFromXml，否则会调用Drawable.createFromResourceStream。我们就只分析createFromResourceStream吧，原理应该大同小异。

```
public static Drawable createFromResourceStream(Resources res, TypedValue value,
            InputStream is, String srcName, BitmapFactory.Options opts) {
    Rect pad = new Rect();

    if (opts == null) opts = new BitmapFactory.Options();

    Bitmap  bm = BitmapFactory.decodeResourceStream(res, value, is, pad, opts);
    if (bm != null) {
        byte[] np = bm.getNinePatchChunk();
        if (np == null || !NinePatch.isNinePatchChunk(np)) {
            np = null;
            pad = null;
        }
        int[] layoutBounds = bm.getLayoutBounds();
        Rect layoutBoundsRect = null;
        if (layoutBounds != null) {
            layoutBoundsRect = new Rect(layoutBounds[0], layoutBounds[1],
                                         layoutBounds[2], layoutBounds[3]);
        }
        return drawableFromBitmap(res, bm, np, pad, layoutBoundsRect, srcName);
    }
    return null;
}
```

眼前一亮，这里出现了传说中的

```
Bitmap  bm = BitmapFactory.decodeResourceStream(res, value, is, pad, opts);
```

所以结论出来了，系统预加载资源时，会给这些Bitmap都decode到内存里，而且是强引用的，所以是GC不掉的。要清理这些内存就必须通过反射清理sPreloadedDrawable，不过要注意适配不同的sdk版本。
