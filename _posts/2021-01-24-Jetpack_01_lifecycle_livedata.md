---
layout: post
title: "Jetpack组件_01_Lifecycle_LiveData"
subtitle: "Jetpack_01_Lifecycle_LiveData"
date: 2021-01-24
author: "hyzhao"
header-img: "img/post-bg-2015.jpg"
tags: ["Jetpack"]
---



# lifecycle

> 生命周期感知型组件可执行操作来响应另一个组件（如 Activity 和 Fragment）的生命周期状态的变化。这些组件有助于您写出更有条理且往往更精简的代码，这样的代码更易于维护。

> 一种常见的模式是在 Activity 和 Fragment 的生命周期方法中实现依赖组件的操作。但是，这种模式会导致代码条理性很差而且会扩散错误。通过使用生命周期感知型组件，您可以将依赖组件的代码从生命周期方法移入组件本身中。

比如在MVP模式中我们要在Presenter中监听Activity的生命周期，从而进行相关的操作。
```java
//伪代码 Activity

publlc void onResume(){
    presenter.onResume()
}
```

虽然此示例看起来没问题，但在真实的应用中，最终会有太多管理界面和其他组件的调用，以响应生命周期的当前状态。管理多个组件会在生命周期方法（如 onStart() 和 onStop()）中放置大量的代码，这使得它们难以维护。

此外，无法保证组件会在 Activity 或 Fragment 停止之前启动。在我们需要执行长时间运行的操作（如 onStart() 中的某种配置检查）时尤其如此。这可能会导致出现一种竞争条件，在这种条件下，onStop() 方法会在 onStart() 之前结束，这使得组件留存的时间比所需的时间要长。

lifecycle 软件包提供的类和接口可帮助您以弹性和隔离的方式解决这些问题。

## 基本用法
我们使用androidX项目
Activity中使用getLifecycle().addObserver()来添加Obsever
```kotlin
class GoodsActivity : BaseActivity<GoodsPresenter<IGoodsView>, IGoodsView>(), IGoodsView {

    lateinit var listView: ListView
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_goods)
        listView = findViewById(R.id.listView)
        presenter.attachView(this)
        presenter.fetch()


    }

    override fun init() {
        super.init()
        //添加observer
        lifecycle.addObserver(presenter)
    }

    override fun createPresenter(): GoodsPresenter<IGoodsView> {
        return GoodsPresenter()
    }

    override fun showGoodsView(goods: MutableList<Goods>) {
        listView.adapter = GoodsAdapter(this, goods)
    }

    override fun showErrorMessage(msg: String) {
    }
}
```
Presenter实现LifecycleObserver接口，使用OnLifecycleEvent注解来订阅

```kotlin
open class BasePresenter<in T : IBaseView> : LifecycleObserver {

    var iGoodsView: WeakReference<*>? = null

    fun attachView(view: T) {
        iGoodsView = WeakReference(view)
    }

    fun detachView() {
        iGoodsView?.clear()
        iGoodsView = null
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    open fun onCreateX(owner: LifecycleOwner) {

    }


    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    open fun onStartX(owner: LifecycleOwner) {

    }
}   
```
## 优点
* Lifecycle可以有效的避免内存泄漏和解决android生命周期的常见难题
* Livecycle   是一个表示android生命周期及状态的对象
* LivecycleOwner  用于连接有生命周期的对象
* LivecycleObserver  用于观察查LifecycleOwner
## 原理分析

AppCompatActivity继承ComponentActivity而ComponentActivity实现了LifecycleOwner接口，并且持有LifecycleRegistry对象


```java
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}

//ComponentActivity.java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        SavedStateRegistryOwner,
        OnBackPressedDispatcherOwner {
        
    //.....
    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    //.....
    
    
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mSavedStateRegistryController.performRestore(savedInstanceState);
        //填入空的Fragment管理生命周期
        ReportFragment.injectIfNeededIn(this);
        if (mContentLayoutId != 0) {
            setContentView(mContentLayoutId);
        }
    }
    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }

}
```
AppCompatActivity 继承的extends androidx.core.app.ComponentActivity中的onCretae方法
ReportFragment.injectIfNeededIn(this);就是在当前的Activity里添加一个ReportFragment
	

```java
@RestrictTo(RestrictTo.Scope.LIBRARY_GROUP_PREFIX)
public class ReportFragment extends Fragment {
    //....
    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatchResume(mProcessListener);
        dispatch(Lifecycle.Event.ON_RESUME);
    }
    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
    //...
}
```

在ReportFragment的各个生命周期函数中调用dispatch方法，而dispatch方法调用LifecycleRegistry的handleLifecycleEvent方法来转发，这样生命周期的状态就会借由LifecycleRegistry通知给各个LifecycleObserver从而调用其中对应Lifecycle.Event的方法。这种通过Fragment来感知Activity生命周期的方法其实在Glide的中也是有体现的。

回到LifecycleRegistry的handleLifecycleEvent方法
```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```
getStateAfter方法获取当前状态的下一个状态,下图为Android Activity 生命周期的状态和事件
![5a819b83be1b2afed824247ed332b490.svg+xml](https://app.yinxiang.com/FileSharing.action?hash=1/b3d09df9ab5e66e392787a0c96c0d83e-198931)@w=1000
moveToState(next)中，将mState=next，再调用sync方法同步状态，看看sync方法

```java

private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            //逆推
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            //正推
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```
backwardPass和forwardPass方法推导的状态可以从上面的图分析到，


```java

private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            //调用各个ObserverWithState的dispatchEvent方法
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
```
ObserverWithState类如下
```java
static class ObserverWithState {
    State mState;
    LifecycleEventObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        //创建LifecycleEventObserver
        mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
//Lifecycling.java
@NonNull
static LifecycleEventObserver lifecycleEventObserver(Object object) {
    boolean isLifecycleEventObserver = object instanceof LifecycleEventObserver;
    boolean isFullLifecycleObserver = object instanceof FullLifecycleObserver;
    if (isLifecycleEventObserver && isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object,
                (LifecycleEventObserver) object);
    }
    if (isFullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object, null);
    }

    if (isLifecycleEventObserver) {
        return (LifecycleEventObserver) object;
    }

    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    //种子
    return new ReflectiveGenericLifecycleObserver(object);
}
```
dispatchEvent方法调用了LifecycleEventObserver的onStateChanged方法，Lifecycling.lifecycleEventObserver方法返回了一个ReflectiveGenericLifecycleObserver对象，


```java
class ReflectiveGenericLifecycleObserver implements LifecycleEventObserver {
    private final Object mWrapped;
    private final CallbackInfo mInfo;

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        //创建CallbackInfo
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        //
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```
ReflectiveGenericLifecycleObserver的构造方法中首先创建CallbackInfo对象mInfo,

看下ClassesInfoCache.sInstance.getInfo(mWrapped.getClass())是如何返回CallbackInfo的
```java
//ClassesInfoCache.java
CallbackInfo getInfo(Class klass) {
    CallbackInfo existing = mCallbackMap.get(klass);
    if (existing != null) {
        return existing;
    }
    existing = createInfo(klass, null);
    return existing;
}
private CallbackInfo createInfo(Class klass, @Nullable Method[] declaredMethods) {
    //获取父类的mHandlerToEvent
    Class superclass = klass.getSuperclass();
    Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
    if (superclass != null) {
        CallbackInfo superInfo = getInfo(superclass);
        if (superInfo != null) {
            handlerToEvent.putAll(superInfo.mHandlerToEvent);
        }
    }

    Class[] interfaces = klass.getInterfaces();
    for (Class intrfc : interfaces) {
        for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
                intrfc).mHandlerToEvent.entrySet()) {
            verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
        }
    }

    Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
    boolean hasLifecycleMethods = false;
    for (Method method : methods) {
        //获取所以带OnLifecycleEvent注解的方法
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation == null) {
            continue;
        }
        hasLifecycleMethods = true;
        Class<?>[] params = method.getParameterTypes();
        int callType = CALL_TYPE_NO_ARG;
        //方法参数大于0
        if (params.length > 0) {
            callType = CALL_TYPE_PROVIDER;
            //第一个参数必须是LifecycleOwner类型
            if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. Must be one and instanceof LifecycleOwner");
            }
        }
        Lifecycle.Event event = annotation.value();
        //校验第二个参数必须是Lifecycle.Event.ON_ANY
        if (params.length > 1) {
            callType = CALL_TYPE_PROVIDER_WITH_EVENT;
            if (!params[1].isAssignableFrom(Lifecycle.Event.class)) {
                throw new IllegalArgumentException(
                        "invalid parameter type. second arg must be an event");
            }
            if (event != Lifecycle.Event.ON_ANY) {
                throw new IllegalArgumentException(
                        "Second arg is supported only for ON_ANY value");
            }
        }
        //方法参数不能大于2
        if (params.length > 2) {
            throw new IllegalArgumentException("cannot have more than 2 params");
        }
        MethodReference methodReference = new MethodReference(callType, method);
        verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
    }
    CallbackInfo info = new CallbackInfo(handlerToEvent);
    mCallbackMap.put(klass, info);
    mHasLifecycleMethods.put(klass, hasLifecycleMethods);
    return info;
}
```

CallbackInfo中的invokeCallbacks通过方式调用Method的invoke方法

```java
static class CallbackInfo {
    final Map<Lifecycle.Event, List<MethodReference>> mEventToHandlers;
    final Map<MethodReference, Lifecycle.Event> mHandlerToEvent;

    CallbackInfo(Map<MethodReference, Lifecycle.Event> handlerToEvent) {
        mHandlerToEvent = handlerToEvent;
        mEventToHandlers = new HashMap<>();
        for (Map.Entry<MethodReference, Lifecycle.Event> entry : handlerToEvent.entrySet()) {
            Lifecycle.Event event = entry.getValue();
            List<MethodReference> methodReferences = mEventToHandlers.get(event);
            if (methodReferences == null) {
                methodReferences = new ArrayList<>();
                mEventToHandlers.put(event, methodReferences);
            }
            methodReferences.add(entry.getKey());
        }
    }

    @SuppressWarnings("ConstantConditions")
    void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
        invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
        invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
                target);
    }

    private static void invokeMethodsForEvent(List<MethodReference> handlers,
            LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
        if (handlers != null) {
            for (int i = handlers.size() - 1; i >= 0; i--) {
                
                handlers.get(i).invokeCallback(source, event, mWrapped);
            }
        }
    }
}

static class MethodReference {
        final int mCallType;
        final Method mMethod;

        MethodReference(int callType, Method method) {
            mCallType = callType;
            mMethod = method;
            mMethod.setAccessible(true);
        }

        //通过方式调用
        void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
            //noinspection TryWithIdenticalCatches
            try {
                switch (mCallType) {
                    case CALL_TYPE_NO_ARG:
                        mMethod.invoke(target);
                        break;
                    case CALL_TYPE_PROVIDER:
                        mMethod.invoke(target, source);
                        break;
                    case CALL_TYPE_PROVIDER_WITH_EVENT:
                        mMethod.invoke(target, source, event);
                        break;
                }
            } catch (InvocationTargetException e) {
                throw new RuntimeException("Failed to call observer method", e.getCause());
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
        }
        //....
}
```
再来看看LifecycleRegistry的addObserver方法

```java
private CallbackInfo createInfo(Class klass, @Nullable Method[] declaredMethods) {}
....
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
            new FastSafeIterableMap<>();
....          
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    //初始化状态
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    //创建ObserverWithState对象
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    //把ObserverWithState放到FastSafeIterableMap中
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
    //第一次放到map时，previous是null
    if (previous != null) {
        return;
    }
    //lifecycleOwner就是前面的Activity
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);//(1)
    mAddingObserverCounter++;
   //
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

流程图如下：
![cfab40026e9ad127df2595be60b8bf66.png](https://app.yinxiang.com/FileSharing.action?hash=1/cfab40026e9ad127df2595be60b8bf66-124083)


# livedata
livedata is a observable

> LiveData 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者

如果观察者（由 Observer 类表示）的生命周期处于 STARTED 或 RESUMED 状态，则 LiveData 会认为该观察者处于活跃状态。LiveData 只会将更新通知给活跃的观察者。为观察 LiveData 对象而注册的非活跃观察者不会收到更改通知。


## 优势
* 确保界面符合数据状态
> LiveData 遵循观察者模式。当生命周期状态发生变化时，LiveData 会通知 Observer 对象。您可以整合代码以在这些 Observer 对象中更新界面。观察者可以在每次发生更改时更新界面，而不是在每次应用数据发生更改时更新界面。

* 不会发生内存泄漏
> 观察者会绑定到 Lifecycle 对象，并在其关联的生命周期遭到销毁后进行自我清理。

* 不会因 Activity 停止而导致崩溃
> 如果观察者的生命周期处于非活跃状态（如返回栈中的 Activity），则它不会接收任何 LiveData 事件。

* 不再需要手动处理生命周期
> 界面组件只是观察相关数据，不会停止或恢复观察。LiveData 将自动管理所有这些操作，因为它在观察时可以感知相关的生命周期状态变化。

* 数据始终保持最新状态
> 如果生命周期变为非活跃状态，它会在再次变为活跃状态时接收最新的数据。例如，曾经在后台的 Activity 会在返回前台后立即接收最新的数据。

* 适当的配置更改
> 如果由于配置更改（如设备旋转）而重新创建了 Activity 或 Fragment，它会立即接收最新的可用数据。

* 共享资源
> 您可以使用单一实例模式扩展 LiveData 对象以封装系统服务，以便在应用中共享它们。LiveData 对象连接到系统服务一次，然后需要相应资源的任何观察者只需观察 LiveData 对象

## 使用LiveData对象

按照以下步骤使用LiveData

1. 创建 LiveData 实例以存储某种类型的数据。这通常在 **ViewModel** 类中完成。
2. 创建可定义 onChanged() 方法的 Observer 对象，该方法可以控制当 LiveData 对象存储的数据更改时会发生什么。通常情况下，您可以在界面控制器（如 Activity 或 Fragment）中创建 Observer 对象。
3. 使用 observe() 方法将 Observer 对象附加到 LiveData 对象。observe() 方法会采用 LifecycleOwner 对象。这样会使 Observer 对象订阅 LiveData 对象，以使其收到有关更改的通知。通常情况下，您可以在界面控制器（如 Activity 或 Fragment）中附加 Observer 对象。


具体示例代码如下:

### 创建 LiveData 对象
LiveData 是一种可用于任何数据的封装容器，其中包括可实现 Collections 的对象，如 List。LiveData 对象通常存储在 ViewModel 对象中，并可通过 getter 方法进行访问，

```java
public abstract class LiveData<T> {
    //...
}
```

LiveData是个抽象类,他的实现类如下，一般我们使用MutableLiveData

![14c125ffe1fb55ad235fbcdb32e27f21.png](https://app.yinxiang.com/FileSharing.action?hash=1/14c125ffe1fb55ad235fbcdb32e27f21-107088)@w=500

```kotlin
class NameViewModel : ViewModel() {
    //Create a LiveData with a String
    val currentName: MutableLiveData<String> by lazy {
        MutableLiveData<String>()
    }
}
```

### 观察 LiveData 对象

在大多数情况下，应用组件的 onCreate() 方法是开始观察 LiveData 对象的正确着手点，原因如下：
* 确保系统不会从 Activity 或 Fragment 的 onResume() 方法进行冗余调用。
* 确保 Activity 或 Fragment 变为活跃状态后具有可以立即显示的数据。一旦应用组件处于 STARTED 状态，就会从它正在观察的 LiveData 对象接收最新值。只有在设置了要观察的 LiveData 对象时，才会发生这种情况。

> 通常，LiveData 仅在数据发生更改时才发送更新，并且仅发送给活跃观察者。此行为的一种例外情况是，观察者从非活跃状态更改为活跃状态时也会收到更新。此外，如果观察者第二次从非活跃状态更改为活跃状态，则只有在自上次变为活跃状态以来值发生了更改时，它才会收到更新。


```kotlin
class NameActivity : AppCompatActivity() {

    //Use the 'by viewModels()' Kotlin property delegate
    //from the activity-ktx artifact
    private val model: NameViewModel by viewModels()

    private var i: Int = 0
    private lateinit var tvText: TextView
    private lateinit var btn: Button
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_name)
        tvText = findViewById(R.id.tvText)
        btn = findViewById(R.id.btn)

        val nameObserver = Observer<String> { newName ->
            //update UI
            tvText.text = newName

        }

        //Observe the LiveData, passing in this activity as the LifecycleOwner and the observer.
        model.currentName.observe(this, nameObserver)

        btn.setOnClickListener {
            i++
            model.currentName.setValue("click $i")
        }
    }
}
```
在传递 nameObserver 参数的情况下调用 observe() 后，系统会立即调用 onChanged()，从而提供 mCurrentName 中存储的最新值。如果 LiveData 对象尚未在 mCurrentName 中设置值，则不会调用 onChanged()。

### 更新 LiveData 对象

设置观察者关系后，您可以更新 LiveData 对象的值（如以下示例中所示），这样当用户点按某个按钮时会触发所有观察者：

```kotlin
btn.setOnClickListener {
    i++
    model.currentName.setValue("click $i")
}
```
在本例中调用 setValue(T) 导致观察者使用值 "click $i" 调用其 onChanged() 方法。本例中演示的是按下按钮的方法，但也可以出于各种各样的原因调用 setValue() 或 postValue() 来更新 mName，这些原因包括响应网络请求或数据库加载完成。在所有情况下，调用 setValue() 或 postValue() 都会触发观察者并更新界面。

您必须调用 setValue(T) 方法以从主线程更新 LiveData 对象。如果在 worker 线程中执行代码，则您可以改用 postValue(T) 方法来更新 LiveData 对象。

## 原理分析

observe()作为入口：

使用LifecycleBoundObserver把观察者和被观察者包装在一起,绑定wrapper作为观察者

```java
//LiveData.java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```
绑定完成后，使用setValue与postValue通知观察者,


```java
//LiveData.java
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

//参数传null和不传null的区别就是如果传null将会通知所有的观察者，反之仅仅通知传入的观察者。
@SuppressWarnings("WeakerAccess") /* synthetic access */
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            //只通知传入的Observer
            considerNotify(initiator);
            initiator = null;
        } else {
            //通知所有Observer
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}


private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //调用mObserver的onChanage()
    observer.mObserver.onChanged((T) mData);
}
```

# LiveDataBus

为了简化LiveData操作，可以仿照EventBus写个LiveDataBus

```java
class LiveDataBus private constructor() {
    //kotlin的一种单例模式
    companion object {
        val instance: LiveDataBus by lazy(mode = LazyThreadSafetyMode.SYNCHRONIZED) {
            LiveDataBus()
        }
    }
    //存放订阅者
    private val bus: MutableMap<String, MutableLiveData<Object>> = mutableMapOf()
    //注册订阅者
    @Synchronized
    fun <T> with(key: String, type: Class<T>): MutableLiveData<T> {
        if (!bus.containsKey(key)) {
            bus[key] = MutableLiveData();
        }
        @Suppress("UNCHECKED_CAST")
        return bus[key] as MutableLiveData<T>
    }
}
```

使用:

```kotlin

//MainActivity.java

Thread {
    (0..10).forEach {
        //发送
        LiveDataBus.instance
            .with("data", String::class.java).postValue("hello $it")
        SystemClock.sleep(1000)
    }
}.start()

```

订阅:

```kotlin
class LiveDataBusTestActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_livedata_bus)
        //订阅
        LiveDataBus.instance
            .with("data", String::class.java).observe(this, {
                Toast.makeText(baseContext, it, Toast.LENGTH_SHORT).show()
            })
    }
}
```
