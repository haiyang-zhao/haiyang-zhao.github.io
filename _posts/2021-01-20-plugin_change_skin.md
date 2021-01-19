---
layout: post
title: "插件化_换肤方案"
subtitle: "插件化_换肤方案"
date: 2021-01-20
author: "hyzhao"
header-img: "img/post-bg-2015.jpg"
tags: ["插件化","换肤"]
---


# 概述

## 实现效果
![1c665e9544d0054692bf3302a23bfda9.gif](https://app.yinxiang.com/FileSharing.action?hash=1/1c665e9544d0054692bf3302a23bfda9-1997785)

## 实现思路

基本流程如下:
![36831aff4b47f3acbfe032014ad6e23e.png](https://app.yinxiang.com/FileSharing.action?hash=1/36831aff4b47f3acbfe032014ad6e23e-20869)


代码地址 [https://github.com/haiyang-zhao/SkinDemo](https://github.com/haiyang-zhao/SkinDemo)
# View收集


第一步要收集需要换肤的控件及其各个属性，从Activity的setContentView()说起
## setContentView()
```java
public class MainActivity extends AppCompatActivity {
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }    

}
```


调用的父类AppCompatActivity的setContentView方法
```java
@Override
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);
}


@NonNull
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        mDelegate = AppCompatDelegate.create(this, this);//返回的是AppCompatDelegateImpl
    }
    return mDelegate;
}
```

最终调用的是AppCompatDelegateImpl的setContentView方法

```java
//AppCompatDelegateImpl.java
@Override
public void setContentView(int resId) {
    ensureSubDecor(); // 1
    ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);//2
    mAppCompatWindowCallback.getWrapped().onContentChanged();
}
```

ensureSubDecor()方法是确保DecorView中添加了subDecor 如下图所示

![751bf732e378b0fbddafd11e4dcf63e0.png](https://app.yinxiang.com/FileSharing.action?hash=1/751bf732e378b0fbddafd11e4dcf63e0-62661)@w=500
然后再调用  LayoutInflater.from(mContext).inflate(resId, contentParent)向subDecor中添加我们自己定义在xml中的view。

## LayoutInflater.inflate()

LayoutInflater.inflate()最终会调用tryCreateView(),tryCreateView方法里调用mFactory2的onCreateView()创建View

```java
//LayoutInflater.java
public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    View view;
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }

    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }

    return view;
}
```

## Hook点mFactory2

可以通过LayoutInflater的setFactory2()方法，来设置自己的Factory2
```java
public void setFactory2(Factory2 factory) {
    if (mFactorySet) {
        throw new IllegalStateException("A factory has already been set on this LayoutInflater");
    }
    if (factory == null) {
        throw new NullPointerException("Given factory can not be null");
    }
    mFactorySet = true;
    if (mFactory == null) {
        mFactory = mFactory2 = factory;
    } else {
        mFactory = mFactory2 = new FactoryMerger(factory, factory, mFactory, mFactory2);
    }
}
```
我们只可以通过反射把mFactorySet设置为false，就可以设置mFactory2了

```java
@SystemService(Context.LAYOUT_INFLATER_SERVICE)
public abstract class LayoutInflater {
    @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.P)
    private boolean mFactorySet;
    @UnsupportedAppUsage
    private Factory2 mFactory2;
    @UnsupportedAppUsage
    private Factory2 mPrivateFactory;
    ...
}

```

把LayoutInflater的mFactory2 Hook成自己定义的Factory2，并在其中完成view的采集工作。

**但是**， Android 10中把mFactory2标注为UnsupportedAppUsage，用户自己的app不能通过反射的方法去Hook了，

这是使用[FreeReflection](https://github.com/tiann/FreeReflection) 可以绕过此限制


## SkinLayoutInflaterFactory

这里 SkinLayoutInflaterFactory还实现Observer，通过观察者默认，一定执行换肤操作，调用其update()，skinAttribute.applySkin()来换肤
```java
public class SkinLayoutInflaterFactory implements LayoutInflater.Factory2, Observer {

    private static final String[] mClassPrefixList = {
            "android.widget.",
            "android.webkit.",
            "android.app.",
            "android.view."
    };


    //记录对应View的构造函数
    private static final HashMap<String, Constructor<? extends View>> mConstructorMap =
            new HashMap<String, Constructor<? extends View>>();

    private static final Class<?>[] mConstructorSignature = new Class[]{
            Context.class, AttributeSet.class};

    // 当选择新皮肤后需要替换View与之对应的属性
    // 页面属性管理器
    private final SkinAttribute skinAttribute;

    // 用于获取窗口的状态框的信息
    private final Activity activity;

    public SkinLayoutInflaterFactory(Activity activity) {
        this.activity = activity;
        this.skinAttribute = new SkinAttribute();
    }


    @Nullable
    @Override

    public View onCreateView(@Nullable View parent, @NonNull String name, @NonNull Context context, @NonNull AttributeSet attrs) {

        //换肤就是在需要时候替换 View的属性(src、background等)
        //所以这里创建 View,从而修改View属性
        View view = createSDKView(name, context, attrs);
        if (null == view) {
            view = createView(name, context, attrs);
        }
        //这就是我们加入的逻辑
        if (null != view) {
            //加载属性
            skinAttribute.lookup(view, attrs);
        }
        return view;
    }

    private View createSDKView(String name, Context context, AttributeSet
            attrs) {
        //如果包含 . 则不是SDK中的view 可能是自定义view包括support库中的View
        if (-1 != name.indexOf('.')) {
            return null;
        }
        //不包含就要在解析的 节点 name前，拼上： android.widget. 等尝试去反射
        for (String s : mClassPrefixList) {
            View view = createView(s + name, context, attrs);
            if (view != null) {
                return view;
            }
        }
        return null;
    }

    private View createView(String name, Context context, AttributeSet
            attrs) {
        Constructor<? extends View> constructor = findConstructor(context, name);
        try {
            return constructor.newInstance(context, attrs);
        } catch (Exception e) {
        }
        return null;
    }


    private Constructor<? extends View> findConstructor(Context context, String name) {
        Constructor<? extends View> constructor = mConstructorMap.get(name);
        if (constructor == null) {
            try {
                Class<? extends View> clazz = context.getClassLoader().loadClass
                        (name).asSubclass(View.class);
                constructor = clazz.getConstructor(mConstructorSignature);
                mConstructorMap.put(name, constructor);
            } catch (Exception e) {
            }
        }
        return constructor;
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull String name, @NonNull Context context, @NonNull AttributeSet attrs) {
        return null;
    }


    @Override
    public void update(Observable o, Object arg) {
        SkinThemeUtils.updateStatusBarColor(activity);
        skinAttribute.applySkin();
    }
}
```

## 何时setFactory2()
再看看MainActivity的onCreate()方法
```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
}
```
我们必须在调用setContentView()之前，super.onCreate()之后去设置值，，为什么要在super.onCreate()之后设呢？

看看 super.onCreate()，即就是AppCompatActivity的onCreate()方法
```java
//AppCompatActivity.java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    final AppCompatDelegate delegate = getDelegate();
    delegate.installViewFactory();//设置setFactory2
    delegate.onCreate(savedInstanceState);
    super.onCreate(savedInstanceState);//3
}

//AppCompatDelegateImpl.java
@Override
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
        LayoutInflaterCompat.setFactory2(layoutInflater, this);
    } else {
        if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImpl)) {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                    + " so we can not install AppCompat's");
        }
    }
}
```
AppCompatActivity的onCreate()中调用AppCompatDelegateImpl的installViewFactory()设置mFactory2的只，所以要保证自己setFactory2()是最后一次调用，否则程序就报错了，

再看看super.onCreate(savedInstanceState)直到Activity的onCreate()方法中，

```java
   protected void onCreate(@Nullable Bundle savedInstanceState) {
        //....
        if (savedInstanceState != null) {
            mAutoFillResetNeeded = savedInstanceState.getBoolean(AUTOFILL_RESET_NEEDED, false);
            mLastAutofillId = savedInstanceState.getInt(LAST_AUTOFILL_ID,
                    View.LAST_APP_AUTOFILL_ID);

            if (mAutoFillResetNeeded) {
                getAutofillManager().onCreate(savedInstanceState);
            }

            Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
            mFragments.restoreAllState(p, mLastNonConfigurationInstances != null
                    ? mLastNonConfigurationInstances.fragments : null);
        }
        mFragments.dispatchCreate();
        dispatchActivityCreated(savedInstanceState);//
        if (mVoiceInteractor != null) {
            mVoiceInteractor.attachActivity(this);
        }
        mRestoredFromBundle = savedInstanceState != null;
        mCalled = true;

    }
    
    
    private void dispatchActivityCreated(@Nullable Bundle savedInstanceState) {
    getApplication().dispatchActivityCreated(this, savedInstanceState);
    Object[] callbacks = collectActivityLifecycleCallbacks();
    if (callbacks != null) {
        for (int i = 0; i < callbacks.length; i++) {
            ((Application.ActivityLifecycleCallbacks) callbacks[i]).onActivityCreated(this,
                    savedInstanceState);
        }
    }
}
```
dispatchActivityCreated()回调了onActivityCreated()接口

因此可以调用Application.registerActivityLifecycleCallbacks()注册Acitity声明周期方法，在其onActivityCreated()中设置mFactory2的值

```java
//注册Activity生命周期的回调
public class SkinActivityLifecycle implements Application.ActivityLifecycleCallbacks {
    private static final String TAG = "SkinActivityLifecycle";
    private SkinManager skinManager;

    public SkinActivityLifecycle(SkinManager skinManager) {

        this.skinManager = skinManager;
    }

    @Override
    public void onActivityCreated(@NonNull Activity activity, @Nullable Bundle savedInstanceState) {
        Log.d(TAG, activity.getClass().getName() + " onActivityCreated");


        SkinThemeUtils.updateStatusBarColor(activity);
        LayoutInflater layoutInflater = activity.getLayoutInflater();

        //利用反射将layoutInflater的私有变量mFactorySet设置为true
        try {
            Field field = LayoutInflater.class.getDeclaredField("mFactorySet");
            field.setAccessible(true);
            field.setBoolean(layoutInflater, false);
        } catch (Exception e) {
            e.printStackTrace();
        }

        SkinLayoutInflaterFactory factory = new SkinLayoutInflaterFactory(activity);
        layoutInflater.setFactory2(factory);

        this.skinManager.addObserver(factory);

    }
    //.....
       @Override
    public void onActivityDestroyed(@NonNull Activity activity) {
        activity.unregisterActivityLifecycleCallbacks(this);
    }
}
```
SkinManager的构造方法中去初始化

```java
public final class SkinManager extends Observable {

    private volatile static SkinManager INSTANCE;

    private final Application app;
    private final SkinActivityLifecycle skinActivityLifecycle;

    private SkinManager(Application app) {
        this.app = app;
        this.skinActivityLifecycle = new SkinActivityLifecycle(this);
        SkinPrefs.init(this.app);
        SkinResources.init(this.app);
        app.registerActivityLifecycleCallbacks(skinActivityLifecycle);


    }
    //...
}    
```

# 获取皮肤包中对应的resource


## 关键API：Resources.getIdentifier()

```java
/**
 * Return a resource identifier for the given resource name.  A fully
 * qualified resource name is of the form "package:type/entry".  The first
 * two components (package and type) are optional if defType and
 * defPackage, respectively, are specified here.
 * 
 * <p>Note: use of this function is discouraged.  It is much more
 * efficient to retrieve resources by identifier than by name.
 * 
 * @param name The name of the desired resource.
 * @param defType Optional default resource type to find, if "type/" is
 *                not included in the name.  Can be null to require an
 *                explicit type.
 * @param defPackage Optional default package to find, if "package:" is
 *                   not included in the name.  Can be null to require an
 *                   explicit package.
 * 
 * @return int The associated resource identifier.  Returns 0 if no such
 *         resource was found.  (0 is not a valid resource ID.)
 */
public int getIdentifier(String name, String defType, String defPackage) {
    return mResourcesImpl.getIdentifier(name, defType, defPackage);
}
```
获取R文件中的资源ID，

```xml
<color name="colorPrimary">#1F1F1F</color>
```
name就是colorPrimary，defType是color，defPackage指定报包名，

如果直接用宿主的Resources对象调用getIdentifier()包名传皮肤包的包名，肯定不能获取到resId，因此要创建包含皮肤包的Resources对象
![20c404deb56c696159a6ba75cc8f0d6c.png](https://app.yinxiang.com/FileSharing.action?hash=1/20c404deb56c696159a6ba75cc8f0d6c-191125)
## 创建皮肤包Resources对象

```java
//宿主res
            Resources appResources = app.getResources();
            try {
                //反射创建assets对象
                AssetManager assetManager = AssetManager.class.newInstance();
                //反射获取addAssetPath方法
                Method method = AssetManager.class.getMethod("addAssetPath", String.class);
                method.invoke(assetManager, path);


                //根据当前的设备显示器信息 与 配置(横竖屏、语言等) 创建Resources
                Resources skinResource = new Resources(assetManager, appResources.getDisplayMetrics
                        (), appResources.getConfiguration());

                //获取外部Apk(皮肤包) 包名
                PackageManager mPm = app.getPackageManager();
                PackageInfo info = mPm.getPackageArchiveInfo(path, PackageManager
                        .GET_ACTIVITIES);
                String packageName = info.packageName;
                SkinResources.getInstance().applySkin(skinResource, packageName);

                //记录
                SkinPrefs.getInstance().setSkin(path);

            } catch (Exception e) {
                e.printStackTrace();
            }
```

这样就能通过name、type和皮肤包的包名，获取皮肤包中对应资源的Id来了

```java

    public int getIdentifier(int resId) {
        if (isDefaultSkin) {
            return resId;
        }

        String resName = mAppResources.getResourceEntryName(resId);
        String resType = mAppResources.getResourceTypeName(resId);

        return mSkinResources.getIdentifier(resName, resType, mSkinPkgName);
    }

```

# 执行换肤


```java
      //切换皮肤
    public void applySkin() {
        applySkinSupport();

        for (SkinPair skinPair : skinPairs) {
            Drawable left = null, top = null, right = null, bottom = null;
            String attrName = skinPair.attrName;
            int resId = skinPair.resId;
            //根据不同的属性设置不同的值
            switch (attrName) {
                case "src":
                    Object background = getInstance().getBackground(resId);
                    //比如@color/xxx
                    if (background instanceof Integer) {
                        view.setBackgroundColor((int) background);
                    }
                    if (background instanceof Drawable) {
                        ViewCompat.setBackground(view, (Drawable) background);
                    }
                    break;
                case "background":
                    background = getInstance().getBackground(skinPair
                            .resId);
                    if (background instanceof Integer) {
                        if (view instanceof ImageView) {
                            ((ImageView) view).setImageDrawable(new ColorDrawable((Integer)
                                    background));
                        }else {
                            view.setBackgroundColor((Integer) background);
                        }
                    }

                    if (background instanceof Drawable) {
                        if (view instanceof ImageView) {
                            ((ImageView) view).setImageDrawable((Drawable) background);
                        } else {

                            view.setBackground((Drawable) background);
                        }
                    }
                    break;
                case "textColor":
                    ((TextView) view).setTextColor(getInstance().getColorStateList
                            (skinPair.resId));
                    break;
                case "drawableLeft":
                    left = getInstance().getDrawable(skinPair.resId);
                    break;
                case "drawableTop":
                    top = getInstance().getDrawable(skinPair.resId);
                    break;
                case "drawableRight":
                    right = getInstance().getDrawable(skinPair.resId);
                    break;
                case "drawableBottom":
                    bottom = getInstance().getDrawable(skinPair.resId);
                    break;
                default:
                    break;
            }

            if (null != left || null != right || null != top || null != bottom) {
                ((TextView) view).setCompoundDrawablesWithIntrinsicBounds(left, top, right,
                        bottom);
            }
        }
    }

    //自定义view支持
    private void applySkinSupport() {
        if (view instanceof SkinViewSupport) {
            ((SkinViewSupport) view).applySkin();
        }
    }
```

# 参考资源

[一种绕过Android P对非SDK接口限制的简单方法](http://weishu.me/2018/06/07/free-reflection-above-android-p/)

[另一种绕过 Android P以上非公开API限制的办法](http://weishu.me/2019/03/16/another-free-reflection-above-android-p/)

[Android Native Hook工具实践](https://gtoad.github.io/2018/07/06/Android-Native-Hook-Practice/)


[可能是全网讲最细的安卓resources.arsc解析教程\(一\)](https://blog.islinjw.cn/2019/05/18/%E5%8F%AF%E8%83%BD%E6%98%AF%E5%85%A8%E7%BD%91%E8%AE%B2%E6%9C%80%E7%BB%86%E7%9A%84%E5%AE%89%E5%8D%93resources-arsc%E8%A7%A3%E6%9E%90%E6%95%99%E7%A8%8B-%E4%B8%80/)
