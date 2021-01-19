---
layout: post
title: "插件化_Hook机制之动态代理"
subtitle: "Android插件化_Hook机制之动态代理"
date: 2021-01-19
author: "Hux"
header-img: "img/post-bg-2015.jpg"
tags: ["Android","插件化","动态代理"]
---

# 代理模式


![67b96d3d5d30eae444171b3aa2209a99.png](https://app.yinxiang.com/FileSharing.action?hash=1/67b96d3d5d30eae444171b3aa2209a99-20324)

## 静态代码
静态代理，是最原始的代理方式；假设我们有一个购物的接口，如下：

```java
public interface Shopping {
    Object[] doShopping(long money);
}
```
它有一个原始的实现，我们可以理解为亲自，直接去商店购物：

```java
public class ShoppingImpl implements Shopping {
    @Override
    public Object[] doShopping(long money) {
        System.out.println("逛淘宝 ,逛商场,买买买!!");
        System.out.println(String.format("花了%s块钱", money));
        return new Object[] { "鞋子", "衣服", "零食" };
    }
}
```

好了，现在我们自己没时间但是需要买东西，于是我们就找了个代理帮我们买：

```java
public class ProxyShopping implements Shopping {

    Shopping base;

    ProxyShopping(Shopping base) {
        this.base = base;
    }

    @Override
    public Object[] doShopping(long money) {

        // 先黑点钱(修改输入参数)
        long readCost = (long) (money * 0.5);

        System.out.println(String.format("花了%s块钱", readCost));

        // 帮忙买东西
        Object[] things = base.doShopping(readCost);

        // 偷梁换柱(修改返回值)
        if (things != null && things.length > 1) {
            things[0] = "被掉包的东西!!";
        }

        return things;
    }
```

很不幸，我们找的这个代理有点坑，坑了我们的钱还坑了我们的货；先忍忍。

## 动态代理

传统的静态代理模式需要为每一个需要代理的类写一个代理类，如果需要代理的类有几百个那不是要累死？为了更优雅地实现代理模式，JDK提供了动态代理方式，可以简单理解为JVM可以在运行时帮我们动态生成一系列的代理类，这样我们就不需要手写每一个静态的代理类了。依然以购物为例，用动态代理实现如下

```java
public static void main(String[] args) {
    Shopping women = new ShoppingImpl();
    // 正常购物
    System.out.println(Arrays.toString(women.doShopping(100)));
    // 招代理
    women = (Shopping) Proxy.newProxyInstance(Shopping.class.getClassLoader(),
            women.getClass().getInterfaces(), new ShoppingHandler(women));

    System.out.println(Arrays.toString(women.doShopping(100)));
}
```

动态代理主要处理InvocationHandler和Proxy类


# 代理Hook

下面我们Hook掉startActivity这个方法，使得每次调用这个方法之前输出一条日志；（当然，这个输入日志有点点弱，只是为了展示原理；只要你想，你想可以替换参数，拦截这个startActivity过程，使得调用它导致启动某个别的Activity，指鹿为马！）

## Hook点 (被Hook的对象)


什么样的对象容易找到？**静态变量和单例**

## Hook startActivity()


startActivity的调用链，找出合适的Hook点,对于Context.startActivity调用的是ConetxtImpl.startActivity方法：

```java
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
    // maintain this for backwards compatibility.
    final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

    if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
            && (targetSdkVersion < Build.VERSION_CODES.N
                    || targetSdkVersion >= Build.VERSION_CODES.P)
            && (options == null
                    || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                        + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                        + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity) null, intent, -1, options);
}
```

这里，实际上使用了ActivityThread类的mInstrumentation成员的execStartActivity方法；注意到，ActivityThread 实际上是主线程，而主线程一个进程只有一个，因此这里是一个良好的Hook点。


```java
@UnsupportedAppUsage
public static ActivityThread currentActivityThread() {
    return sCurrentActivityThread;
}
```
接下来就是想要Hook掉我们的主线程对象，也就是把这个主线程对象里面的mInstrumentation给替换成我们修改过的代理对象；要替换主线程对象里面的字段，首先我们得拿到主线程对象的引用，如何获取呢？ActivityThread类里面有一个静态方法currentActivityThread可以帮助我们拿到这个对象类；但是ActivityThread是一个隐藏类，我们需要用反射去获取，代码如下：


```java
// 先获取到当前的ActivityThread对象
Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
currentActivityThreadMethod.setAccessible(true);
Object currentActivityThread = currentActivityThreadMethod.invoke(null);
```

拿到这个currentActivityThread之后，我们需要修改它的mInstrumentation这个字段为我们的代理对象，我们先实现这个代理对象，由于JDK动态代理只支持接口，而这个Instrumentation是一个类，没办法，我们只有手动写静态代理类，覆盖掉原始的方法即可

```java
/**
 * Instrumentation的静态代理类
 */
public class EvilInstrumentation extends Instrumentation {

    private static final String TAG = "EvilInstrumentation";
    private Instrumentation mBase;

    public EvilInstrumentation(Instrumentation mBase) {
        this.mBase = mBase;
    }


    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        Log.d(TAG, "execStartActivity is called.");

        try {
            Method method = mBase.getClass().getDeclaredMethod("execStartActivity",
                    Context.class, IBinder.class, IBinder.class, Activity.class, Intent.class,
                    int.class, Bundle.class);

            method.invoke(mBase, who, contextThread, token, target, intent, requestCode, options);

        } catch (Exception e) {
            e.printStackTrace();
        }
        //
        return null;
    }
}
```

Ok，有了代理对象，我们要做的就是偷梁换柱！代码比较简单，采用反射直接修改：



```java
public static void attach() {
    try {
        //获取ActivityThread的Class对象
        Class<?> clazz = Class.forName("android.app.ActivityThread");
        Method method = clazz.getDeclaredMethod("currentActivityThread");

        method.setAccessible(true);
        //currentActivityThread是一个static函数所以可以直接invoke，不需要带实例参数
        Object currentActivityThread = method.invoke(null);

        //Hook ActivityThread中的 mInstrumentation

        Field field = clazz.getDeclaredField("mInstrumentation");
        field.setAccessible(true);

        //获取ActivityThread中的 mInstrumentation
        Instrumentation fieldValue = (Instrumentation) field.get(currentActivityThread);
        field.set(currentActivityThread,new EvilInstrumentation(fieldValue));

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
好了，我们启动一个Activity测试一下，结果如下：

![9a591180c5b605449806ebc2e087065d.png](https://app.yinxiang.com/FileSharing.action?hash=1/9a591180c5b605449806ebc2e087065d-20123)
可见，Hook确实成功了！这就是使用代理进行Hook的原理——偷梁换柱。整个Hook过程简要总结如下：

* 1、 寻找Hook点，原则是静态变量或者单例对象，尽量Hook pulic的对象和方法，非public不保证每个版本都一样，需要适配。
* 2、选择合适的代理方式，如果是接口可以用动态代理；如果是类可以手动写代理也可以使用cglib。
* 3、偷梁换柱——用代理对象替换原始对象