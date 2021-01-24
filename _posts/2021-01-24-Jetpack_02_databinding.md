---
layout: post
title: "Jetpack组件_02_Databinding"
subtitle: "Jetpack组件_02_Databinding"
date: 2021-01-24
author: "hyzhao"
header-img: "img/post-bg-2015.jpg"
tags: ["Jetpack"]
---



#  简介
> 数据绑定库是一种支持库，借助该库，您可以使用声明性格式（而非程序化地）将布局中的界面组件绑定到应用中的数据源。


布局通常是使用调用界面框架方法的代码在 Activity 中定义的。例如，以下代码调用 findViewById() 来查找 TextView 微件并将其绑定到 viewModel 变量的 userName 属性：

```kotlin
findViewById<TextView>(R.id.sample_text).apply {
    text = viewModel.userName
}
```
以下示例展示了如何在布局文件中使用数据绑定库将文本直接分配到微件。这样就无需调用上述任何 Java 代码。请注意赋值表达式中 @{} 语法的使用：

```xml
<TextView android:text="@{viewmodel.userName}" />
```
#  gradle配置

```gradle
 defaultConfig {
    ....
    dataBinding {
        enabled true
    }
 }
```

# 基本使用

1. 定义实体对象

```kotlin
class User(name: String, age: Int) : BaseObservable() {

    @Bindable
    var name: String = name
        set(value) {
            field = value
            notifyPropertyChanged(BR.name)
        }


    @Bindable
    var age: Int = age
        set(value) {
            field = value
            notifyPropertyChanged(BR.age)
        }
}
```
2. 布局文件中使用
activity_main.xml
根节点用layout元素，并添加data节点，声明要使用的实体对象
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

        <variable
            name="user"
            type="com.zhy.databinding.User" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity">

        <TextView
            android:id="@+id/tv_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{user.name}"
            app:layout_constraintBottom_toTopOf="@+id/tv_age"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintVertical_chainStyle="packed" />


        <TextView
            android:id="@+id/tv_age"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@{String.valueOf(user.age)}"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/tv_name" />

        <Button
            android:id="@+id/btn"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/tv_age"
            android:text="点击测试"
            />

        <Button
            android:layout_marginTop="10dp"
            android:id="@+id/btn2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/btn"
            android:text="单个属性变化"
            />
    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```
3. 代码中使用
```kotlin
class MainActivity : AppCompatActivity() {

    private var age: Int = 18
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityMainBinding = DataBindingUtil.setContentView(
            this, R.layout.activity_main
        )
        var user = User("张三", age++)
        binding.user = user

        binding.btn.setOnClickListener {
            //对象改变
            user = User("张三", age++)
            binding.user = user
        }

        binding.btn2.setOnClickListener {
            //对象属性改变
            age++
            user.age = age
            user.name = "张三$age"
        }
    }
}
```

#   核心原理

##  layout 布局文件分离

编译时会根据activity_main.xml生成两份xml文件
* build/intermediates/data_binding_layout_info_type_merge/debug/out/activity_main-layout.xml

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Layout directory="layout" filePath="app/src/main/res/layout/activity_main.xml" isBindingData="true" isMerge="false" layout="activity_main" modulePackage="com.zhy.databinding" rootNodeType="androidx.constraintlayout.widget.ConstraintLayout">
	<Variables declared="true" name="user" type="com.zhy.databinding.User">
		<location endLine="9" endOffset="45" startLine="7" startOffset="8"/>
	</Variables>
	<Targets>
		<Target tag="layout/activity_main_0" view="androidx.constraintlayout.widget.ConstraintLayout">
			<Expressions/>
			<location endLine="59" endOffset="55" startLine="12" startOffset="4"/>
		</Target>
		<Target id="@+id/tv_name" tag="binding_1" view="TextView">
			<Expressions>
				<Expression attribute="android:text" text="user.name">
					<Location endLine="21" endOffset="38" startLine="21" startOffset="12"/>
					<TwoWay>false</TwoWay>
					<ValueLocation endLine="21" endOffset="36" startLine="21" startOffset="28"/>
				</Expression>
			</Expressions>
			<location endLine="26" endOffset="63" startLine="17" startOffset="8"/>
		</Target>
		<Target id="@+id/tv_age" tag="binding_2" view="TextView">
			<Expressions>
				<Expression attribute="android:text" text="String.valueOf(user.age)">
					<Location endLine="33" endOffset="53" startLine="33" startOffset="12"/>
					<TwoWay>false</TwoWay>
					<ValueLocation endLine="33" endOffset="51" startLine="33" startOffset="28"/>
				</Expression>
			</Expressions>
			<location endLine="37" endOffset="64" startLine="29" startOffset="8"/>
		</Target>
		<Target id="@+id/btn" view="Button">
			<Expressions/>
			<location endLine="47" endOffset="13" startLine="39" startOffset="8"/>
		</Target>
		<Target id="@+id/btn2" view="Button">
			<Expressions/>
			<location endLine="58" endOffset="13" startLine="49" startOffset="8"/>
		</Target>
	</Targets>
</Layout>
```
每个View都对应一个Target元素，Target记录了view的id,tag,以及使用的表达式(使用@{})等等信息


* build/intermediates/incremental/mergeDebugResources/stripped.dir/layout/activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>

                                                       
                                                   

    

                 
                       
                                              
           

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity" android:tag="layout/activity_main_0" xmlns:android="http://schemas.android.com/apk/res/android" xmlns:app="http://schemas.android.com/apk/res-auto" xmlns:tools="http://schemas.android.com/tools">

        <TextView
            android:id="@+id/tv_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:tag="binding_1"    
            app:layout_constraintBottom_toTopOf="@+id/tv_age"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            app:layout_constraintVertical_chainStyle="packed" />


        <TextView
            android:id="@+id/tv_age"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:tag="binding_2"                   
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintLeft_toLeftOf="parent"
            app:layout_constraintRight_toRightOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/tv_name" />

        <Button
            android:id="@+id/btn"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/tv_age"
            android:text="点击测试"
            />

        <Button
            android:layout_marginTop="10dp"
            android:id="@+id/btn2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@+id/btn"
            android:text="单个属性变化"
            />
    </androidx.constraintlayout.widget.ConstraintLayout>
```

这个xml会剔除掉layout节点，并且为绑定了@{}的View添加一个tag

## 源码分析

首先看apt生成的类

![6468c19fcab299cfcda2f3847e95aeab.png](https://app.yinxiang.com/FileSharing.action?hash=1/6468c19fcab299cfcda2f3847e95aeab-89883)

类图如下:

![297aa8265830b304536747f02a8a95c7.png](https://app.yinxiang.com/FileSharing.action?hash=1/297aa8265830b304536747f02a8a95c7-38316)

### 入口1： DataBindingUtil.setContentView

```kotlin
//DataBindingUtil.java
public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
        int layoutId) {
    return setContentView(activity, layoutId, sDefaultComponent);
}
```

setContentView方法重载：

```java
public static <T extends ViewDataBinding> T setContentView(@NonNull Activity activity,
        int layoutId, @Nullable DataBindingComponent bindingComponent) {
    activity.setContentView(layoutId);
    View decorView = activity.getWindow().getDecorView();
    ViewGroup contentView = (ViewGroup) decorView.findViewById(android.R.id.content);
    return bindToAddedViews(bindingComponent, contentView, 0, layoutId);
}
```
从这个方法可以看出activity和xml还是通过setContentView(layoutId)来进行关联的，

再看bindToAddedViews方法

```java
private static <T extends ViewDataBinding> T bindToAddedViews(DataBindingComponent component,
        ViewGroup parent, int startChildren, int layoutId) {
    final int endChildren = parent.getChildCount();
    final int childrenAdded = endChildren - startChildren;
    if (childrenAdded == 1) {
        final View childView = parent.getChildAt(endChildren - 1);
        return bind(component, childView, layoutId);
    } else {
        final View[] children = new View[childrenAdded];
        for (int i = 0; i < childrenAdded; i++) {
            children[i] = parent.getChildAt(i + startChildren);
        }
        return bind(component, children, layoutId);
    }
}

static <T extends ViewDataBinding> T bind(DataBindingComponent bindingComponent, View[] roots,
        int layoutId) {
    return (T) sMapper.getDataBinder(bindingComponent, roots, layoutId);
}
```

最终调用到sMapper.getDataBinder，sMapper又是什么呢？

```java
private static DataBinderMapper sMapper = new DataBinderMapperImpl();
```

最后的最后会调到DataBinderMapperImpl的getDataBinder方法。

```java
//MergedDataBinderMapper.java
@Override
public ViewDataBinding getDataBinder(DataBindingComponent bindingComponent, View[] view,
        int layoutId) {
    for(DataBinderMapper mapper : mMappers) {
        ViewDataBinding result = mapper.getDataBinder(bindingComponent, view, layoutId);
        if (result != null) {
            return result;
        }
    }
    if (loadFeatures()) {
        return getDataBinder(bindingComponent, view, layoutId);
    }
    return null;
}
```
DataBinderMapperImpl的构造方法中把apt生成的com.zhy.databinding.DataBinderMapperImpl类添加到mMapper集合中
```java
//androidx.databinding.DataBinderMapperImpl.java
public class DataBinderMapperImpl extends MergedDataBinderMapper {
  DataBinderMapperImpl() {
    addMapper(new com.zhy.databinding.DataBinderMapperImpl());
  }
}
```

最终调用的是com.zhy.databinding.DataBinderMapperImpl.getDataBinder()方法

```java
public class DataBinderMapperImpl extends DataBinderMapper {
  private static final int LAYOUT_ACTIVITYMAIN = 1;

  private static final SparseIntArray INTERNAL_LAYOUT_ID_LOOKUP = new SparseIntArray(1);

  static {
    INTERNAL_LAYOUT_ID_LOOKUP.put(com.zhy.databinding.R.layout.activity_main, LAYOUT_ACTIVITYMAIN);
  }

  @Override
  public ViewDataBinding getDataBinder(DataBindingComponent component, View view, int layoutId) {
    int localizedLayoutId = INTERNAL_LAYOUT_ID_LOOKUP.get(layoutId);
    if(localizedLayoutId > 0) {
      final Object tag = view.getTag();
      if(tag == null) {
        throw new RuntimeException("view must have a tag");
      }
      switch(localizedLayoutId) {
        case  LAYOUT_ACTIVITYMAIN: {
          if ("layout/activity_main_0".equals(tag)) {
            return new ActivityMainBindingImpl(component, view);
          }
          throw new IllegalArgumentException("The tag for activity_main is invalid. Received: " + tag);
        }
      }
    }
    return null;
  }
  //......
}  
```

build/intermediates/incremental/mergeDebugResources/stripped.dir/layout/activity_main.xml这个文件中找到tag为layout/activity_main_0的View ->ConstraintLayout,最后返回 ActivityMainBindingImpl(component, view)

```xml
  <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        tools:context=".MainActivity" android:tag="layout/activity_main_0" xmlns:android="http://schemas.android.com/apk/res/android" xmlns:app="http://schemas.android.com/apk/res-auto" xmlns:tools="http://schemas.android.com/tools">
       //..... 
```

继续看apt生成的ActivityMainBindingImpl构造方法:

```java
public ActivityMainBindingImpl(@Nullable androidx.databinding.DataBindingComponent bindingComponent, @NonNull View root) {
    this(bindingComponent, root, mapBindings(bindingComponent, root, 5, sIncludes, sViewsWithIds));
}
```

继续查看mapBindings方法：

```java
    /**
     * Walks the view hierarchy under root and pulls out tagged Views, includes, and views with
     * IDs into an Object[] that is returned. This is used to walk the view hierarchy once to find
     * all bound and ID'd views.
     *
     * @param bindingComponent The binding component to use with this binding.
     * @param root The root of the view hierarchy to walk.
     * @param numBindings The total number of ID'd views, views with expressions, and includes
     * @param includes The include layout information, indexed by their container's index.
     * @param viewsWithIds Indexes of views that don't have tags, but have IDs.
     * @return An array of size numBindings containing all Views in the hierarchy that have IDs
     * (with elements in viewsWithIds), are tagged containing expressions, or the bindings for
     * included layouts.
     * @hide
     */
    protected static Object[] mapBindings(DataBindingComponent bindingComponent, View root,
            int numBindings, IncludedLayouts includes, SparseIntArray viewsWithIds) {
        Object[] bindings = new Object[numBindings];
        mapBindings(bindingComponent, root, bindings, includes, viewsWithIds, true);
        return bindings;
    }
```

翻译下注释：
> 在根目录下遍历视图层次结构，并将带有ID的标记视图，包含视图和ID放入返回的Object []中。 这用于遍历视图层次结构一次以查找所有绑定视图和ID视图。

mapBindings()遍历root节点下的所有带tag的view。

再看看另外一个重载的构造方法:

```java
private ActivityMainBindingImpl(androidx.databinding.DataBindingComponent bindingComponent, View root, Object[] bindings) {
    super(bindingComponent, root, 1
        , (android.widget.Button) bindings[3]
        , (android.widget.Button) bindings[4]
        , (android.widget.TextView) bindings[2]
        , (android.widget.TextView) bindings[1]
        );
    this.mboundView0 = (androidx.constraintlayout.widget.ConstraintLayout) bindings[0];
    this.mboundView0.setTag(null);
    this.tvAge.setTag(null);
    this.tvName.setTag(null);
    setRootTag(root);
    // listeners
    invalidateAll();
}
```
这样就解析所有的findViewById的View了
至此，可以看到xml中申明的view会在解析阶段存储在ActivityMainBindingImpl中，这也就是为什么能在Activity中通过binding直接访问View。


### 入口2： binding.user = user

调用的是ActivityMainBindingImpl.setUser方法
```java
//com.zhy.databinding.databinding.ActivityMainBindingImpl.java
public void setUser(@Nullable com.zhy.databinding.User User) {
    updateRegistration(0, User);
    this.mUser = User;
    synchronized(this) {
        mDirtyFlags |= 0x1L;
    }
    notifyPropertyChanged(BR.user);
    super.requestRebind();
}
```

继续看下updateRegistration(0, User);


```java

protected boolean updateRegistration(int localFieldId, Observable observable) {
    return updateRegistration(localFieldId, observable, CREATE_PROPERTY_LISTENER);
}
    
```
CREATE_PROPERTY_LISTENER的代码如下：为一个CreateWeakListener常量
```java
private static final CreateWeakListener CREATE_PROPERTY_LISTENER = new CreateWeakListener() {
    @Override
    public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
        return new WeakPropertyListener(viewDataBinding, localFieldId).getListener();
    }
};

```

继续看updateRegistration(localFieldId, observable, CREATE_PROPERTY_LISTENER）方法

```java
private boolean updateRegistration(int localFieldId, Object observable,
        CreateWeakListener listenerCreator) {
    if (observable == null) {
        return unregisterFrom(localFieldId);
    }
    WeakListener listener = mLocalFieldObservers[localFieldId];
    if (listener == null) {
        registerTo(localFieldId, observable, listenerCreator);
        return true;
    }
    if (listener.getTarget() == observable) {
        return false;//nothing to do, same object
    }
    unregisterFrom(localFieldId);
    registerTo(localFieldId, observable, listenerCreator);
    return true;
}
```

此方法就是把CreateWeakListener 和数据的id形成绑定关系，数据的Id可通过BR文件内容获得，BR文件内容如下：

```java
package com.zhy.databinding;

public class BR {
  public static final int _all = 0;

  public static final int age = 1;

  public static final int name = 2;

  public static final int user = 3;
}
```

name和age是因为我们重写了set方法，并加入了@Bindable注解，而user 是定义在xml中的，都会在BR文件中生成对应的ID，然后继续看notifyPropertyChanged方法：

```java
public void notifyPropertyChanged(int fieldId) {
    synchronized (this) {
        if (mCallbacks == null) {
            return;
        }
    }
    mCallbacks.notifyCallbacks(this, fieldId, null);
}
```
这个方法就跟踪到这，后面会继续讨论，接下来看super.requestRebind方法:

```java
protected void requestRebind() {
    if (mContainingBinding != null) {
        mContainingBinding.requestRebind();
    } else {
        final LifecycleOwner owner = this.mLifecycleOwner;
        if (owner != null) {
            Lifecycle.State state = owner.getLifecycle().getCurrentState();
            if (!state.isAtLeast(Lifecycle.State.STARTED)) {
                return; // wait until lifecycle owner is started
            }
        }
        synchronized (this) {
            if (mPendingRebind) {
                return;
            }
            mPendingRebind = true;
        }
        if (USE_CHOREOGRAPHER) {
            mChoreographer.postFrameCallback(mFrameCallback);
        } else {
            mUIThreadHandler.post(mRebindRunnable);
        }
    }
}
```

可以看到最后需要Android版本，但是最终都会调用到mRebindRunnable：

```java
/**
 * Runnable executed on animation heartbeat to rebind the dirty Views.
 */
private final Runnable mRebindRunnable = new Runnable() {
    @Override
    public void run() {
        synchronized (this) {
            mPendingRebind = false;
        }
        processReferenceQueue();

        if (VERSION.SDK_INT >= VERSION_CODES.KITKAT) {
            // Nested so that we don't get a lint warning in IntelliJ
            if (!mRoot.isAttachedToWindow()) {
                // Don't execute the pending bindings until the View
                // is attached again.
                mRoot.removeOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                mRoot.addOnAttachStateChangeListener(ROOT_REATTACHED_LISTENER);
                return;
            }
        }
        executePendingBindings();
    }
};
```

然后继续看executePendingBindings：

```java
/**
 * Evaluates the pending bindings, updating any Views that have expressions bound to
 * modified variables. This <b>must</b> be run on the UI thread.
 */
public void executePendingBindings() {
    if (mContainingBinding == null) {
        executeBindingsInternal();
    } else {
        mContainingBinding.executePendingBindings();
    }
}



/**
 * Evaluates the pending bindings without executing the parent bindings.
 */
private void executeBindingsInternal() {
    if (mIsExecutingPendingBindings) {
        requestRebind();
        return;
    }
    if (!hasPendingBindings()) {
        return;
    }
    mIsExecutingPendingBindings = true;
    mRebindHalted = false;
    if (mRebindCallbacks != null) {
        mRebindCallbacks.notifyCallbacks(this, REBIND, null);

        // The onRebindListeners will change mPendingHalted
        if (mRebindHalted) {
            mRebindCallbacks.notifyCallbacks(this, HALTED, null);
        }
    }
    if (!mRebindHalted) {
        executeBindings();
        if (mRebindCallbacks != null) {
            mRebindCallbacks.notifyCallbacks(this, REBOUND, null);
        }
    }
    mIsExecutingPendingBindings = false;
}
```

这里最终会调用到executeBindings方法，但他是一个抽象方法，所以看他的实现就好了：

```java
   @Override
    protected void executeBindings() {
        long dirtyFlags = 0;
        synchronized(this) {
            dirtyFlags = mDirtyFlags;
            mDirtyFlags = 0;
        }
        java.lang.String userName = null;
        int userAge = 0;
        com.zhy.databinding.User user = mUser;
        java.lang.String stringValueOfUserAge = null;

        if ((dirtyFlags & 0xfL) != 0) {


            if ((dirtyFlags & 0xbL) != 0) {

                    if (user != null) {
                        // read user.name
                        userName = user.getName();
                    }
            }
            if ((dirtyFlags & 0xdL) != 0) {

                    if (user != null) {
                        // read user.age
                        userAge = user.getAge();
                    }


                    // read String.valueOf(user.age)
                    stringValueOfUserAge = java.lang.String.valueOf(userAge);
            }
        }
        // batch finished
        if ((dirtyFlags & 0xdL) != 0) {
            // api target 1

            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.tvAge, stringValueOfUserAge);
        }
        if ((dirtyFlags & 0xbL) != 0) {
            // api target 1

            androidx.databinding.adapters.TextViewBindingAdapter.setText(this.tvName, userName);
        }
    }
```

到这里应该就都能看的懂了，就是通过一层层回调以后最终还是会通过setText来更新UI。

### 入口3：notifyPropertyChanged(BR.age)

调用user.age = age时，会调用user.setAge()方法，会调用notifyPropertyChanged(BR.age)

```java
public void notifyPropertyChanged(int fieldId) {
    synchronized (this) {
        if (mCallbacks == null) {
            return;
        }
    }
    mCallbacks.notifyCallbacks(this, fieldId, null);
}
```


接上前面的，单数据发生变更时，都会调用 mCallbacks.notifyCallbacks：

```java
public synchronized void notifyCallbacks(T sender, int arg, A arg2) {
    mNotificationLevel++;
    notifyRecurse(sender, arg, arg2);
    mNotificationLevel--;
    if (mNotificationLevel == 0) {
        if (mRemainderRemoved != null) {
            for (int i = mRemainderRemoved.length - 1; i >= 0; i--) {
                final long removedBits = mRemainderRemoved[i];
                if (removedBits != 0) {
                    removeRemovedCallbacks((i + 1) * Long.SIZE, removedBits);
                    mRemainderRemoved[i] = 0;
                }
            }
        }
        if (mFirst64Removed != 0) {
            removeRemovedCallbacks(0, mFirst64Removed);
            mFirst64Removed = 0;
        }
    }
}
```

中间调用 notifyRecurse()方法


```java
   private void notifyRecurse(T sender, int arg, A arg2) {
        final int callbackCount = mCallbacks.size();
        final int remainderIndex = mRemainderRemoved == null ? -1 : mRemainderRemoved.length - 1;

        // Now we've got all callbakcs that have no mRemainderRemoved value, so notify the
        // others.
        notifyRemainder(sender, arg, arg2, remainderIndex);

        // notifyRemainder notifies all at maxIndex, so we'd normally start at maxIndex + 1
        // However, we must also keep track of those in mFirst64Removed, so we add 2 instead:
        final int startCallbackIndex = (remainderIndex + 2) * Long.SIZE;

        // The remaining have no bit set
        notifyCallbacks(sender, arg, arg2, startCallbackIndex, callbackCount, 0);
    }
    
    
private void notifyCallbacks(T sender, int arg, A arg2, final int startIndex,
        final int endIndex, final long bits) {
    long bitMask = 1;
    for (int i = startIndex; i < endIndex; i++) {
        if ((bits & bitMask) == 0) {
            mNotifier.onNotifyCallback(mCallbacks.get(i), sender, arg, arg2);
        }
        bitMask <<= 1;
    }
}    
```

mNotifier.onNotifyCallback是一个抽象方法，所以看他的实现类PropertyChangeRegistry，
然后继续调用WeakPropertyListener的onPropertyChanged方法：


```java
//WeakPropertyListener.java
@Override
public void onPropertyChanged(Observable sender, int propertyId) {
    ViewDataBinding binder = mListener.getBinder();
    if (binder == null) {
        return;
    }
    Observable obj = mListener.getTarget();
    if (obj != sender) {
        return; // notification from the wrong object?
    }
    binder.handleFieldChange(mListener.mLocalFieldId, sender, propertyId);
}
```

又回调到ViewDataBind的handleFieldChange方法：

```java

private void handleFieldChange(int mLocalFieldId, Object object, int fieldId) {
    if (mInLiveDataRegisterObserver) {
        // We're in LiveData registration, which always results in a field change
        // that we can ignore. The value will be read immediately after anyway, so
        // there is no need to be dirty.
        return;
    }
    boolean result = onFieldChange(mLocalFieldId, object, fieldId);
    if (result) {
        requestRebind();
    }
}
```

最终调用requestRebind

### registerTo注册监听器

现在看下registerTo是如何为User的属性注册监听器的

```java
//ViewDataBinding.java
protected void registerTo(int localFieldId, Object observable,
        CreateWeakListener listenerCreator) {
    if (observable == null) {
        return;
    }
    WeakListener listener = mLocalFieldObservers[localFieldId];
    if (listener == null) {
        listener = listenerCreator.create(this, localFieldId);
        mLocalFieldObservers[localFieldId] = listener;
        if (mLifecycleOwner != null) {
            listener.setLifecycleOwner(mLifecycleOwner);
        }
    }
    listener.setTarget(observable);
}
```

此时localFieldId是0,对应BR中的_all表示所有BR中的值，listenerCreator是CREATE_PROPERTY_LISTENER对象，它的create方法如下.

```java
private static final CreateWeakListener CREATE_PROPERTY_LISTENER = new CreateWeakListener() {
    @Override
    public WeakListener create(ViewDataBinding viewDataBinding, int localFieldId) {
        return new WeakPropertyListener(viewDataBinding, localFieldId).getListener();
    }
};
```

看看WeakPropertyListener,getListener()返回的是WeakPropertyListener对象

```java
 private static class WeakPropertyListener extends Observable.OnPropertyChangedCallback
            implements ObservableReference<Observable> {
    final WeakListener<Observable> mListener;

    public WeakPropertyListener(ViewDataBinding binder, int localFieldId) {
        mListener = new WeakListener<Observable>(binder, localFieldId, this);
    }
    
    @Override
    public WeakListener<Observable> getListener() {
        return mListener;
    }
    //...
}
```
getListener()又返回WeakListener对象，

```java
//androidx.databinding.ViewDataBinding.WeakListener.java
private static class WeakListener<T> extends WeakReference<ViewDataBinding> {
    private final ObservableReference<T> mObservable;
    protected final int mLocalFieldId;
    private T mTarget;

    public WeakListener(ViewDataBinding binder, int localFieldId,
            ObservableReference<T> observable) {
        super(binder, sReferenceQueue);
        mLocalFieldId = localFieldId;
        mObservable = observable;
    }

    public void setLifecycleOwner(LifecycleOwner lifecycleOwner) {
        mObservable.setLifecycleOwner(lifecycleOwner);
    }

    public void setTarget(T object) {
        unregister();
        mTarget = object;
        if (mTarget != null) {
            mObservable.addListener(mTarget);
        }
    }
    //...     
}        
```

registerTo最后调用了setTarget()方法，mObservable.addListener(mTarget)，mObervable就是传过来的WeakPropertyListener，mTarget是我们传过来的User对象


```java
//WeakPropertyListener.java
@Override
public void addListener(Observable target) {
    target.addOnPropertyChangedCallback(this);
}
```

再到BaseObservable中

```java
//BaseObservable.java
@Override
public void addOnPropertyChangedCallback(@NonNull OnPropertyChangedCallback callback) {
    synchronized (this) {
        if (mCallbacks == null) {
            mCallbacks = new PropertyChangeRegistry();
        }
    }
    mCallbacks.add(callback);
}

```
这样就把WeakPropertyListener监听器注册到mCallbacks(PropertyChangeRegistry)中

# 时序图

![ae3aea370e5fe5505827d9ece0f2b148.png](https://app.yinxiang.com/FileSharing.action?hash=1/ae3aea370e5fe5505827d9ece0f2b148-375349)



requestRebind()最终会重新刷新界面
