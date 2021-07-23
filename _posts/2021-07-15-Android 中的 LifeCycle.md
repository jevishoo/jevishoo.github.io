---
layout:     post
title:      Android 中的 LifeCycle
subtitle:   
date:       2021-07-15
author:     JH
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Android
    - Java
---

### 学习之前
在正式阅读源码之前，很有必要先介绍三个名词：**LifecycleOwner** ，**LifecycleObserver**，**Lifecycle** 。

`LifecycleOwner` 是一个接口，接口通常用来声明具备某种能力，`LifecycleOwner` 的能力就是**具有生命周期**。`Activity` 和 `Fragment` 就是典型的生命周期组件。
我们也可以自定义生命周期组件，而 `LifecycleOwner` 就提供了 `getLifecycle()` 方法来获取 `Lifecycle` 对象。

```java
/**
 * A class that has an Android lifecycle. These events can be used by custom components to
 * handle lifecycle changes without implementing any code inside the Activity or the Fragment.
 */
@SuppressWarnings({"WeakerAccess", "unused"})
public interface LifecycleOwner {
    /**
     * Returns the Lifecycle of the provider.
     *
     * @return The lifecycle of the provider.
     */
    @NonNull
    Lifecycle getLifecycle();
}
```

`LifecycleObserver` 是**生命周期观察者**，为空接口。它没有任何方法，而是依赖 `OnLifecycleEvent` 注解来接收生命周期回调。

```java
/**
 * Marks a class as a LifecycleObserver. It does not have any methods, instead, relies on
 * OnLifecycleEvent annotated methods.
 */
@SuppressWarnings("WeakerAccess")
public interface LifecycleObserver {

}
```

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface OnLifecycleEvent {
    Lifecycle.Event value();
}
```

`Lifecycle` 是生命周期组件和生命周期观察者之间的桥梁。`Lifecycle` 是具体的生命周期对象，每个 `LifecycleOwner` 都会持有 `Lifecycle` 。通过 `Lifecycle` 可以获取当前生命周期状态，添加/删除 生命周期观察者等等。

`Lifecycle` 内部定义了两个枚举类，`Event` 和 `State` 。

- `Event` 表示生命周期事件，与 `LifecycleOwner` 的生命周期事件是相对应的。

  ```java
  @SuppressWarnings("WeakerAccess")
  public enum Event {
    /**
     * Constant for onCreate event of the LifecycleOwner.
     */
    ON_CREATE,
    /**
     * Constant for onStart event of the LifecycleOwner.
     */
    ON_START,
    /**
     * Constant for onResume event of the LifecycleOwner.
     */
    ON_RESUME,
    /**
     * Constant for onPause event of the LifecycleOwner.
     */
    ON_PAUSE,
    /**
     * Constant for onStop event of the LifecycleOwner.
     */
    ON_STOP,
    /**
     * Constant for onDestroy event of the LifecycleOwner}.
     */
    ON_DESTROY,
    /**
     * An Event constant that can be used to match all events.
     */
    ON_ANY
  }
  ```

- `State` 表示生命周期状态

  ```java
  /**
   * Lifecycle states. You can consider the states as the nodes in a graph and
   * {@link Event}s as the edges between these nodes.
   */
  @SuppressWarnings("WeakerAccess")
  public enum State {
    /**
     * Destroyed state for a LifecycleOwner. After this event, this Lifecycle will not dispatch
     * any more events. For instance, for an {@link android.app.Activity}, this state is reached
     * right before Activity's onDestroy call.
     */
    DESTROYED,
  
    /**
     * Initialized state for a LifecycleOwner. For an {@link android.app.Activity}, this is
     * the state when it is constructed but has not received onCreate yet.
     */
    INITIALIZED,
  
    /**
     * Created state for a LifecycleOwner. For an {@link android.app.Activity}, this state
     * is reached in two cases:
     *     <li>after onCreate call;
     *     <li>right before onStop call.
     */
    CREATED,
  
    /**
     * Started state for a LifecycleOwner. For an {@link android.app.Activity}, this state
     * is reached in two cases:
     *     <li>after onStart call;
     *     <li>right before onPause call.
     */
    STARTED,
  
    /**
     * Resumed state for a LifecycleOwner. For an {@link android.app.Activity}, this state
     * is reached after onResume is called.
     */
    RESUMED;
  
    /**
     * Compares if this State is greater or equal to the given state.
     *
     * @param state State to compare with
     * @return true if this State is greater or equal to the given state
     */
    public boolean isAtLeast(@NonNull State state) {
      return compareTo(state) >= 0;
    }
  }
  ```

  `State` 可能相对比较难以理解，特别是其中枚举值的顺序：`DESTROYED → INITIALIZED → CREATED → STARTED → RESUMED`

**总结**：生命周期组件 `LifecycleOwner` 在进入特定的生命周期后，发送特定的生命周期事件 `Event` ，通知 `Lifcycle` 进入特定的 `State` ，进而回调生命周期观察者 `LifeCycleObserver` 的指定方法。



### LifecycleObserver

`LifecycleObserver` 有子类 `FullLifecycleObserver`、`GenericLifecycleObserver`

使用者可以视情况选择 `LifecycleObserver`、`FullLifecycleObserver`、`GenericLifecycleObserver`, 但框架内部都会将其转换为 `GenericLifecycleObserver` 或其子类（ `FullLifecycleObserverAdapter`、`ReflectiveGenericLifecycleObserver`、`SingleGeneratedAdapterObserver` 、`CompositeGeneratedAdaptersObserver` ）。 其转换行为在类 `Lifecycling` 中：

```java
@NonNull
static GenericLifecycleObserver getCallback(final Object object) {
    final LifecycleEventObserver observer = lifecycleEventObserver(object);
    return new GenericLifecycleObserver() {
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                                   @NonNull Lifecycle.Event event) {
          	observer.onStateChanged(source, event);
        }
    };
}

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
    return new ReflectiveGenericLifecycleObserver(object);
}
```

**注意**：如果引用了注解生成器`kapt "android.arch.lifecycle:compiler:1.1.1"`, 则会生成相应的 `GeneratedAdapter `子类。反之就不会有相应的生成类。因此，如果 `resolveObserverCallbackType()` 方法有找到生成类，则采用代码生成方式，否则采取反射调用。

- ReflectiveGenericLifecycleObserver 反射实现

  ```java
  class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {  
      private final Object mWrapped;
      private final CallbackInfo mInfo;
  
      ReflectiveGenericLifecycleObserver(Object wrapped) {
          mWrapped = wrapped;
          mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
      }
  
      @Override
      public void onStateChanged(LifecycleOwner source, Event event) {
          mInfo.invokeCallbacks(source, event, mWrapped);
      }
  }
  ```

  关键部分为 CallbackInfo 的构造：`ClassesInfoCache.sInstance.getInfo(mWrapped.getClass())`

  首先看下 CallbackInfo 的结构：

  ```java
  // ClassesIndoCache.java
  
  static class CallbackInfo {  
      // 事件 -> 方法列表
      final Map<Lifecycle.Event, List<MethodReference>> mEventToHandlers;
      // 方法 -> 事件
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
          // 事件分发与处理
          invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
          invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
                  target);
      }
  
      private static void invokeMethodsForEvent(List<MethodReference> handlers,
      LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
          if (handlers != null) {
              for (int i = handlers.size() - 1; i >= 0; i--) {
                  // 遍历，调用 MethodReference.invokeCallback
                  handlers.get(i).invokeCallback(source, event, mWrapped);
              }
          }
      }
  }
  ```

  CallbackInfo 实质是将这个类的注解事件与方法提取出来，做了一个双向 map。而 `ReflectiveGenericLifecycleObserver` 中初始化就是得到此 CallbackInfo ：

  ```java
  CallbackInfo getInfo(Class klass) {  
      // 先读取缓存
      CallbackInfo existing = mCallbackMap.get(klass);
      if (existing != null) {
          return existing;
      }
      // create
      existing = createInfo(klass, null);
      return existing;
  }
  
  private CallbackInfo createInfo(Class klass, @Nullable Method[] declaredMethods) {  
      // 先提取父类的 CallbackInfo
      Class superclass = klass.getSuperclass();
      Map<MethodReference, Lifecycle.Event> handlerToEvent = new HashMap<>();
      if (superclass != null) {
          CallbackInfo superInfo = getInfo(superclass);
          if (superInfo != null) {
              handlerToEvent.putAll(superInfo.mHandlerToEvent);
          }
      }
  
      // 再提取接口的 CallbackInfo
      Class[] interfaces = klass.getInterfaces();
      for (Class intrfc : interfaces) {
          for (Map.Entry<MethodReference, Lifecycle.Event> entry : getInfo(
                  intrfc).mHandlerToEvent.entrySet()) {
          // verifyAndPutHandler 的作用是实现了接口或者覆写了父类的方法，但是添加了不同的注解事件。
          verifyAndPutHandler(handlerToEvent, entry.getKey(), entry.getValue(), klass);
      }
  
      // 最后处理类自身的注解
      Method[] methods = declaredMethods != null ? declaredMethods : getDeclaredMethods(klass);
      boolean hasLifecycleMethods = false;
      // 遍历方法，寻找被 OnLifecycleEvent 注解的方法
      for (Method method : methods) {
          OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
          if (annotation == null) {
              continue;
          }
          hasLifecycleMethods = true;
          // 处理参数个数，最多两个参数
          Class<?>[] params = method.getParameterTypes();
          int callType = CALL_TYPE_NO_ARG;
          if (params.length > 0) {
              callType = CALL_TYPE_PROVIDER;
              if (!params[0].isAssignableFrom(LifecycleOwner.class)) {
                          throw new IllegalArgumentException(
                                  "invalid parameter type. Must be one and instanceof LifecycleOwner");
                      }
          }
          Lifecycle.Event event = annotation.value();
  
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
          if (params.length > 2) {
              throw new IllegalArgumentException("cannot have more than 2 params");
          }
          MethodReference methodReference = new MethodReference(callType, method);
          verifyAndPutHandler(handlerToEvent, methodReference, event, klass);
      }
      // new CallbackInfo
      CallbackInfo info = new CallbackInfo(handlerToEvent);
      // 放入缓存中
      mCallbackMap.put(klass, info);
      mHasLifecycleMethods.put(klass, hasLifecycleMethods);
      return info;
  }
  ```

  

### 添加观察者

```java
/**
 * Adds a LifecycleObserver that will be notified when the LifecycleOwner changes state.
 * The given observer will be brought to the current state of the LifecycleOwner.
 * For example, if the LifecycleOwner is in STARTED state, the given observer will receive Event.ON_CREATE events.
 *
 * @param observer The observer to notify.
 */
@MainThread
public abstract void addObserver(@NonNull LifecycleObserver observer);
```

`addObserver` 方法是 `Lifcycle` 抽象类的抽象方法，由代码可以看出就是给当前 `Lifcycle` 对象添加生命周期观察者。

```kotlin
lifecycle.addObserver(mLifecycleObserver)

private val mLifecycleObserver = object : LifecycleObserver() {
  	override fun onCreate(owner: LifecycleOwner?) {  }

  	override fun onStart(owner: LifecycleOwner?) {  }

  	override fun onResume(owner: LifecycleOwner?) {  }

  	override fun onPause(owner: LifecycleOwner?) {  }

  	override fun onStop(owner: LifecycleOwner?) {  }

  	override fun onDestroy(owner: LifecycleOwner?) {  }
}
```

此 `lifecycle` 是其实就是 `LifecycleOwner` 中的 `getLifecycle()` 返回的对象。需要注意的是 `AppCompatActivity` 并没有实现 `LifecycleOwner`，其父类  `FragmentActivity` 也没有，而它的爷爷类 `ComponentActivity` 中才实现了 `LifecycleOwner` 接口，看一下接口的实现：

```java
@NonNull
@Override
public Lifecycle getLifecycle() {
  	return mLifecycleRegistry;
}

private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
```

返回的 `mLifecycleRegistry` 是 `LifecycleRegistry` 对象，`LifecycleRegistry` 是 `Lifecycle` 的实现类，所以，生命周期对象就是这个 `LifecycleRegistry`。该类重写的 `addObserver` 方法就是添加生命周期观察者的方法。

```java
// LifecycleRegistry.java
  
// 保存 LifecycleObserver 及其对应的 State
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
        new FastSafeIterableMap<>();

// 当前生命周期状态
private State mState;

@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
  	// ObserverWithState 是一个静态内部类，后述
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
      	return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

  	// 判断是否重入
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
  
  	// 引入 parentState, targetState 等都是为了保证列表中后加的 Observer 的状态不能大于前面的， 这样做之后，如果列表第一个和最后一个的状态和 LifecycleRegistry.mState 相等时，就说明状态同步完成了。
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
  
  	// 如果观察者的初始状态小于 targetState ，则同步到 targetState
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
      
      	// 状态转移
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

添加观察者对象需要注意所谓的“重入问题”，就是 `addObserver` 会触发 Observer 生命周期函数的调用，而 Observer 在生命周期函数中又调用了 `addObserver` 等方法。

“倒灌问题”，即你在 `onResume()` 中调用 `addObserver()` 方法来添加观察者，观察者依然可以依次接收到 `onCreate` 和 `onStart` 事件 ，最终同步到 `targetState` 。这个 `targetState` 是通过 `calculateTargetState(observer)` 方法计算出来的。

```java
/**
 * 计算出的 targetState 一定是小于等于当前 mState 的
 */
private State calculateTargetState(LifecycleObserver observer) {
  	// 获取当前 Observer 的前一个 Observer
  	Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

  	State siblingState = previous != null ? previous.getValue().mState : null;
    // 无重入情况下可不考虑 parentState ，为 null
    State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
    : null;
  	return min(min(mState, siblingState), parentState);
}
```

我们可以添加多个生命周期观察者，这时候就需要注意维护它们的状态。每次添加新的观察者的初始状态是 `INITIALIZED` ，需要把它同步到当前生命周期状态，确切的说，同步到一个不大于当前状态的 `targetState` 。从源码中的计算方式也有所体现，`targetState` 是 **当前状态 mState**，**mObserverMap 中最后一个观察者的状态** ，**有重入情况下 parentState 的状态** 这三者中的最小值。

为什么要取这个最小值呢？我是这么理解的，当有新的生命周期事件时，需要将 `mObserverMap` 中的所有观察者都同步到新的同一状态，这个同步过程可能尚未完成，所以新加入的观察者只能先同步到最小状态。注意在 `addObserver` 方法的 `while` 循环中，新的观察者每改变一次生命周期，都会调用 `calculateTargetState()` 重新计算 `targetState` 。

- 新加入的观察者的状态同步补充

  `LifecycleRegistry` 给出的思路是：程序保证后加入 map 的 Observer 状态不能大于 map 前面的状态，在这个前提下，如果 map 中第一个 Observer 和最后一个 Observer 的状态相等，并且等于 lifecycle 的状态时，就说明完成同步。

  如何保证后加入 map 的 Observer 不能大于 map 前面的状态呢？

  1. `addObserver` 时调用 `calculateTargetState(observer)`。 取 map 中前一个的状态、parentState、自身 state 的前者。
  2. parentState 是为了应对重入问题， 重入的 `addObserver` 状态不能走到外层 `addObserver` 前面去。
  3. 状态更改分发时，如果状态减小，则 map 从后往前遍历执行分发；如果状态增加，则 map 从前往后遍历执行分发。

最终的稳定状态下，没有生命周期切换，没有添加新的观察者，`mObserverMap` 中的所有观察者应该处于同一个生命周期状态。



### 分发生命周期事件

观察者已经添加完成了，那么如何将生命周期的变化通知观察者呢？

再回到 `ComponentActivity` ，里面并没有重写所有的生命周期函数，而是在 `onCreate()` 加入了一行代码：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
  	super.onCreate(savedInstanceState);
  	ReportFragment.injectIfNeededIn(this);
}
```

```java
public static void injectIfNeededIn(Activity activity) {
    if (Build.VERSION.SDK_INT >= 29) {
        // On API 29+, we can register for the correct Lifecycle callbacks directly
        activity.registerActivityLifecycleCallbacks(
          new LifecycleCallbacks());
    }
    // Prior to API 29 and to maintain compatibility with older versions of
    // ProcessLifecycleOwner (which may not be updated when lifecycle-runtime is updated and
    // need to support activities that don't extend from FragmentActivity from support lib),
    // use a framework fragment to get the correct timing of Lifecycle events
    android.app.FragmentManager manager = activity.getFragmentManager();
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        // Hopefully, we are the first to make a transaction.
        manager.executePendingTransactions();
    }
}
```

这里向 Activity 注入了一个没有页面的 Fragment ，通过注入 Fragment 来代理权限请求。不出意外，`ReportFragment` 才是真正分发生命周期的地方。

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}

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

@Override
public void onPause() {
    super.onPause();
    dispatch(Lifecycle.Event.ON_PAUSE);
}

@Override
public void onStop() {
    super.onStop();
    dispatch(Lifecycle.Event.ON_STOP);
}

@Override
public void onDestroy() {
    super.onDestroy();
    dispatch(Lifecycle.Event.ON_DESTROY);
    // just want to be sure that we won't leak reference to an activity
    mProcessListener = null;
}
```

先关注 `dispatch()` 方法：

```java
static void dispatch(@NonNull Activity activity, @NonNull Lifecycle.Event event) {
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
```

在`ReportFragment` 的各个生命周期函数中通过 `dispatch()` 方法来分发生命周期事件， 然后调用  `LifecycleRegistry` 的 `handleLifecycleEvent()` 方法来处理 。这里假定现在要经历从 `onStart()` 同步到 `onResume()` 的过程，即`handleLifecycleEvent()` 方法中的参数是  `ON_RESUME` 。

```java
// 设置当前状态并通知观察者
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```

`getStateAfter()` 的作用是根据 Event 获取事件之后处于的状态 ，并通知观察者同步到此生命周期状态。

```java
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
```

参数是 `ON_RESUME` ，所以需要同步到的状态是 `RESUMED` 。接下来看看 `moveToState()` 方法的逻辑。

```java
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

首先将要同步到的生命周期状态赋给当前生命周期状态 `mState` ，此时 `mState` 的值就是 `RESUMED` 。然后调用 `sync()` 方法同步所有观察者的状态。

```java
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                + "new events from it.");
        return;
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // mState 是当前状态，如果 mState 小于 mObserverMap 中的状态值，调用 backwardPass()
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        // 如果 mState 大于 mObserverMap 中的状态值，调用 forwardPass()
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

这里会比较 `mState` 和  `mObserverMap` 中观察者的 `State` 值，判断是需要向前还是向后同步状态。现在 `mState` 的值是 `RESUMED` , 而观察者还停留在上一状态 `STARTED` ，所以观察者的状态都得往前挪一步，这里调用的是 `forwardPass()` 方法。

```java
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        // 向上传递事件，直到 observer 的状态值等于当前状态值
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            // 分发生命周期事件
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}
```

`forwardPass()` 会同步 `mObserverMap` 中的所有观察者到指定生命周期状态，如果跨度比较大，会依次分发中间状态。分发生命周期事件最终依赖 `ObserverWithState` 的 `dispatchEvent()` 方法。

上面假定的场景是 `ON_START` 到 `ON_RESUME` 的过程。现在假定另一个场景，我直接按下 Home 键返回桌面，当前 Activity 的生命周期从`onResumed` 到 `onPaused` ，流程如下。

1. `ReportFragment`  调用 `dispatch(Lifecycle.Event.ON_PAUSE)` ，分发 `ON_PAUSE` 事
2. 调用 `LifecycleRegistry.handleLifecycleEvent()` 方法，参数是 `ON_PAUSE`
3. `getStateAfter()` 得到要同步到的状态是  `STARTED` ，并赋给 `mState`，接着调用 `moveToState()`
4. `moveToState(Lifecycle.State.STARTED)` 中调用 `sync()` 方法同步
5. `sync()` 方法中，`mState` 的值是 `STARTED` ，而 `mObserverMap` 中观察者的状态都是 `RESUMED` 。所以观察者们都需要往后挪一步，这调用的就是 `backwardPass()` 方法。

`backwardPass()` 方法其实和 `forwardPass()` 差不多。

```java
private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        // 向下传递事件，直到 observer 的状态值等于当前状态值
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            // 分发生命周期事件
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
```

二者唯一的区别就是获取要分发的事件，一个是 `upEvent()` ，一个是 `downEvent()` 。

`upEvent()` 是获取 state 升级所需要经历的事件，`downEvent()` 是获取 state 降级所需要经历的事件。

```java
private static Event upEvent(State state) {
    switch (state) {
        case INITIALIZED:
        case DESTROYED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        case RESUMED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}

private static Event downEvent(State state) {
    switch (state) {
        case INITIALIZED:
            throw new IllegalArgumentException();
        case CREATED:
            return ON_DESTROY;
        case STARTED:
            return ON_STOP;
        case RESUMED:
            return ON_PAUSE;
        case DESTROYED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

从 `STARTED` 到 `RESUMED` 需要升级，`upEvent(STARTED)` 的返回值是 `ON_RESUME` 。 从 `RESUMED` 到 `STARTED` 需要降级，`downEvent(RESUMED)`的返回值是 `ON_PAUSE` 。

看到这不知道你有没有一点懵，State 和 Event  的关系我也摸索了很长一段时间才理清楚。首先还记得 `State` 的枚举值顺序吗？

`DESTROYED → INITIALIZED → CREATED → STARTED → RESUMED`

`DESTROYED` 最小，`RESUMED` 最大 。`onResume` 进入到 `onPause` 阶段最后分发的生命周期事件的确是 `ON_PAUSE` ，但是将观察者的状态置为了 `STARTED` 。

关于 `State` 和 `Event` 的关系，说明图如下所所示：

![img_lifecycle](https://raw.githubusercontent.com/jevishoo/jevishoo.github.io/master/img/android_lifecycle.png)

### 回调注解方法

同步 Observer 生命周期的 `sync()` 方法最终会调用 `ObserverWithState` 的 `dispatchEvent()` 方法。

```java
static class ObserverWithState {
    State mState;
    GenericLifecycleObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.getCallback(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        // ReflectiveGenericLifecycleObserver.onStateChanged()
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

`mLifecycleObserver` 通过 `Lifecycling.getCallback()` 方法赋值。

```java
@NonNull
static GenericLifecycleObserver getCallback(Object object) {
    if (object instanceof FullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object);
    }

    if (object instanceof GenericLifecycleObserver) {
        return (GenericLifecycleObserver) object;
    }

    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    // 获取 type
    // GENERATED_CALLBACK 表示注解生成的代码
    // REFLECTIVE_CALLBACK 表示使用反射
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
    return new ReflectiveGenericLifecycleObserver(object);
}
```

如果使用的是 `DefaultLifecycleObserver` ，而 `DefaultLifecycleObserver` 又是继承 `FullLifecycleObserver` 的，所以这里会返回 `FullLifecycleObserverAdapter` 。

如果只是普通的 `LifecycleObserver` ，那么就需要通过 `getObserverConstructorType()` 方法判断使用的是注解还是反射。

```java
private static int getObserverConstructorType(Class<?> klass) {
    if (sCallbackCache.containsKey(klass)) {
        return sCallbackCache.get(klass);
    }
    int type = resolveObserverCallbackType(klass);
    sCallbackCache.put(klass, type);
    return type;
}

private static int resolveObserverCallbackType(Class<?> klass) {
    // anonymous class bug:35073837
    // 匿名内部类使用反射
    if (klass.getCanonicalName() == null) {
        return REFLECTIVE_CALLBACK;
    }

    // 寻找注解生成的 GeneratedAdapter 类
    Constructor<? extends GeneratedAdapter> constructor = generatedConstructor(klass);
    if (constructor != null) {
        sClassToAdapters.put(klass, Collections
                .<Constructor<? extends GeneratedAdapter>>singletonList(constructor));
        return GENERATED_CALLBACK;
    }

    // 寻找被 OnLifecycleEvent 注解的方法
    boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
    if (hasLifecycleMethods) {
        return REFLECTIVE_CALLBACK;
    }

    // 没有找到注解生成的 GeneratedAdapter 类，也没有找到 OnLifecycleEvent 注解，
    // 则向上寻找父类
    Class<?> superclass = klass.getSuperclass();
    List<Constructor<? extends GeneratedAdapter>> adapterConstructors = null;
    if (isLifecycleParent(superclass)) {
        if (getObserverConstructorType(superclass) == REFLECTIVE_CALLBACK) {
            return REFLECTIVE_CALLBACK;
        }
        adapterConstructors = new ArrayList<>(sClassToAdapters.get(superclass));
    }

    // 寻找是否有接口实现
    for (Class<?> intrface : klass.getInterfaces()) {
        if (!isLifecycleParent(intrface)) {
            continue;
        }
        if (getObserverConstructorType(intrface) == REFLECTIVE_CALLBACK) {
            return REFLECTIVE_CALLBACK;
        }
        if (adapterConstructors == null) {
            adapterConstructors = new ArrayList<>();
        }
        adapterConstructors.addAll(sClassToAdapters.get(intrface));
    }
    if (adapterConstructors != null) {
        sClassToAdapters.put(klass, adapterConstructors);
        return GENERATED_CALLBACK;
    }

    return REFLECTIVE_CALLBACK;
}
```

注意其中的 `hasLifecycleMethods()` 方法：

```java
boolean hasLifecycleMethods(Class klass) {
    if (mHasLifecycleMethods.containsKey(klass)) {
        return mHasLifecycleMethods.get(klass);
    }

    Method[] methods = getDeclaredMethods(klass);
    for (Method method : methods) {
        OnLifecycleEvent annotation = method.getAnnotation(OnLifecycleEvent.class);
        if (annotation != null) {
            createInfo(klass, methods);
            return true;
        }
    }
    mHasLifecycleMethods.put(klass, false);
    return false;
}
```

这里会去寻找 `OnLifecycleEvent` 注解。所以我们通过 `OnLifecycleEvent`  注解实现的 `MyObserver` 的类型是 `REFLECTIVE_CALLBACK` ，表示使用反射调用。注意另一个类型 `GENERATED_CALLBACK` 表示使用注解生成的代码，而不是反射。

所以，**Lifecycle 可以选择使用 apt 编译期生成代码来避免使用运行时反射，以优化性能**，想到了 **EventBus 的索引加速** 默认也是关闭的。

我们自己定义的在普通的观察者调用的是 `ReflectiveGenericLifecycleObserver.onStateChanged()` 

```java
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    private final Object mWrapped; // Observer 对象
    private final CallbackInfo mInfo; // 反射获取注解信息

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        // 调用 ClassesInfoCache.CallbackInfo.invokeCallbacks()
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```

再追进 `ClassesInfoCache.CallbackInfo.invokeCallbacks()` 方法。

```java
void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
    // 不仅分发了当前生命周期事件，还分发了 ON_ANY
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
```

即反射调用 `OnLifecycleEvent` 注解标记的生命周期回调方法。