---
layout:     post
title:      Android 中的 EventBus
subtitle:   
date:       2021-07-20
author:     JH
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Android
    - Java
---

### 学习之前
先说EventBus是什么： **EventBus是 基于 订阅/发布 模式实现的 基于事件的异步分发处理系统**。好处就是能够解耦 订阅者 和 发布者，简化代码

### 案例展示

```kotlin
data class MsgEvent(val msg: String, val code: Int) 

class MainActivity : AppCompatActivity() {
    companion object{
        val TAG = "EventBusDemo"
    }
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        EventBus.getDefault().register(this)

        welcome.setOnClickListener {
            EventBus.getDefault().post(MsgEvent("Hello, EventBus", 1))
        }
    }

    @Subscribe(priority = 1)
    fun onHandlerMsg(event: MsgEvent){
        Log.e(TAG, "MainActivity # onHandlerMsg  ${event.msg} - ${event.code}")
    }

    @Subscribe(priority = 2)
    fun onHandlerMsg2(event: MsgEvent){
        Log.e(TAG, "MainActivity # onHandlerMsg2  ${event.msg} - ${event.code}")
    }

    override fun onDestroy() {
        super.onDestroy()
        if(EventBus.getDefault().isRegistered(this)){
            EventBus.getDefault().unregister(this)
        }
    }
}
```

当按钮点击时，Log日志：

```shell
2021-07-19 20:20:02.987 21273-21273/com.jevis.helloworld E/MainActivity: Jevis - MainActivity # onHandlerMsg1  Jevis - Hello, EventBus - 1
2021-07-19 20:20:02.987 21273-21273/com.jevis.helloworld E/MainActivity: Jevis - MainActivity # onHandlerMsg  Jevis - Hello, EventBus - 1
```



### go

### 订阅

`EventBus.getDefault().register(this)` 就会注册 MainActivity对象 为订阅者。 这个过程分为两步：

1. 获取 `MainActivity` 对象中方法有 `@Subscribe` 修饰且参数**有且仅有**一个的方法列表。
2. 把这些信息记录起来，以供后续发送者发送消息时通知使用。

```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    // 1.获取订阅者有 @Subscribe 修饰且参数有且仅有一个的方法的列表。
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    // 2.把这些信息记录起来，以供后续发送者发送消息时通知使用。
    synchronized (this) {
      for (SubscriberMethod subscriberMethod : subscriberMethods) {
        	subscribe(subscriber, subscriberMethod);
      }
    }
}
```

#### 方法列表的获取

获取 `MainActivity` 对象中方法有 `@Subscribe` 修饰且参数有且仅有一个的方法列表，有两种实现方式：**反射** 和 **APT**

- 反射

  ```java
  methods = findState.clazz.getDeclaredMethods();
  
  for (Method method : methods) {
    	int modifiers = method.getModifiers();
    	if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
        	Class<?>[] parameterTypes = method.getParameterTypes();
        	if (parameterTypes.length == 1) {
            	Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
            	if (subscribeAnnotation != null) {
                	Class<?> eventType = parameterTypes[0];
                  if (findState.checkAdd(method, eventType)) {
                    ThreadMode threadMode = subscribeAnnotation.threadMode();
                    findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                                                         subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                  }
                }
            }
        }
    }
  ```

  可以看到找出对应方法后，将这些方法相关的信息都封装到 `SubscriberMethod` 类中，也就是说**这个类有订阅者时间处理方法的所有信息**。

  ```java
  public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
          this.method = method;  // 方法
          this.threadMode = threadMode;  // 线程模式
          this.eventType = eventType;  // 事件
          this.priority = priority;  // 优先级
          this.sticky = sticky;  // 是否粘性
      }
  ```

- APT 见后


#### 信息记录

上一步骤得到的这些信息是如何存的呢，这里要讲个映射表数据结构 `subscriptionsByEventType` :

- `key`：事件类型，如本例 MsgEvent.class
- `value`：一个按照订阅方法优先级排序的订阅者的列表集合

**subscriptionsByEventType** 这个集合很重要，存储了**订阅事件**到**所有该事件订阅者**的一个映射。之后发送消息的时候，会直接从这里取出所有订阅了此事件的订阅者，依次通知,完成事件的分发。

这里有还有一个类需要提一下，`Subscription` 代表了订阅者。包含**订阅者对象**和**订阅者的事件订阅方法**。

```java
final class Subscription {
    final Object subscriber;  // 订阅者对象
    final SubscriberMethod subscriberMethod;  // 订阅者的某个方法
  	volatile boolean active;
  	
  	Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        active = true;
    }
}
```

整个订阅过程完毕。同样的`unregister`过程，就是删除 `subscriptionsByEventType` 中对应的订阅者即可。



### 发布

```kotlin
EventBus.getDefault().post(MsgEvent("Hello, EventBus", 1))
```

发布消息的时候，会从 `subscriptionsByEventType` 中找到所有的 `Subscription`，然后挨个通知。我们看核心的通知方法 `postToSubscription` :

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
              	// temporary: technically not correct as poster not decoupled from subscriber
             		invokeSubscriber(subscription, event);
            }
        		break;
        case BACKGROUND:
            if (isMainThread) {
              	backgroundPoster.enqueue(subscription, event);
            } else {
              	invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
        		throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}

```

可以看到不同的线程模式下（默认是 `POSTING` ）都会调用到 `invokeSubscriber()` 方法来通过反射调用对应方法：

```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
      	subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
      	handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
      	throw new IllegalStateException("Unexpected exception", e);
    }
}
```

值得注意的是，`ThreadMode ` 是用于标识最终被 `EventBus` 调用时该方法所在的线程，每一个订阅者的订阅方法（有 `@Subscribe` 修饰的方法）都有一个 `ThreadMode` 的概念，模式分为：

- `POSTING` 默认的线程模式。订阅者的订阅方法将被调用的线程 和 post 事件时所在的线程一致，通过反射直接调用。这个方式会阻塞 `posting` thread，所以尽量避免做一些耗时的操作，因为有可能阻塞的是主线程。

- `MAIN`  如果发送事件时在Android的主线程，则订阅者将被直接调用（`blocking`）。 如果发送事件时不在Android 主线程，则会把事件放入一个队列，等待挨个处理（`not-blocking`）。

- `MAIN_ORDERED`  和 `MAIN` 不一样的是，它总是通过 Android的 `Handler` 机制把事件包装成消息，放入主线程的消息队列。它总是 `not-blocing` 的

- `BACKGROUND` 代表执行订阅者订阅方法总是在子线程中。如果 post 事件所在的线程是子线程，则就在当前线程执行订阅者的订阅方法（`blocking`）；如果调用post 事件所在的线程是主线程，会开启一个线程，执行订阅者订阅方法（有用到线程池）(`not-blocking`)。

  虽然是通过线程池开启的线程， `BACKGROUND` 还是会尽量尝试在开启的线程中多处理几次发送的事件，这样即一次尽可能的使用线程的资源。如果在此线程从事件队列里取事件分发时，一直有事件塞进事件队列的话，则它就会一直循环处理事件的分发。

- `ASYNC`  也是表示订阅者的订阅方法会在子线程中处理。 和 `BACKGROUND` 不一样的是，无论如何它每次都会把事件包装在一个新线程中去执行（`not-blocking`）（这句话有瑕疵，因为是通过线程池控制的，所以运行时难免是线程池中缓存的线程）

看一个简单的 `Poster` 实现 `AsyncPoster` ，即 `ASYNC`对应的 `Poster` ：

```java
class AsyncPoster implements Runnable, Poster {

    private final PendingPostQueue queue;
    private final EventBus eventBus;

    AsyncPoster(EventBus eventBus) {
        this.eventBus = eventBus;
        queue = new PendingPostQueue();
    }

    public void enqueue(Subscription subscription, Object event) {
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        queue.enqueue(pendingPost);
        eventBus.getExecutorService().execute(this);  // 线程池
    }

    @Override
    public void run() {
        PendingPost pendingPost = queue.poll();
        if(pendingPost == null) {
            throw new IllegalStateException("No pending post available");
        }
        eventBus.invokeSubscriber(pendingPost);
    }
}
```

`AsyncPoster` 是每当有事件发送到消息队列中时，都会使用线程池开启一个子线程，去处理这段耗时的操作。

保存事件的 **事件队列**的设计如下：

```java
final class PendingPostQueue {
    private PendingPost head;
    private PendingPost tail;

  //入队列
  synchronized void enqueue(PendingPost pendingPost) {}
  //出队列
  synchronized PendingPost poll() {}
}
```

`PendingPostQueue` 是有链表组成的队列，保存了 `head` 和 `tail` 引用方便入队、出队的操作。 此队列中的数据元素是 `PendingPost` 类型，封装了 `event` 实体 和  `Subscription`订阅者实体

```java
final class PendingPost {
    // 是一个静态的List，保存了回收的 `PendingPost`
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();
    Object event;  // 事件
    Subscription subscription;  // 订阅者
    PendingPost next;  // 构成队列的关键
  
  	/**
  	 * 获取 PendingPostPool 中的队尾的 PendingPost
  	 */
  	static PendingPost obtainPendingPost(Subscription subscription, Object event) {}

  	/**
  	 * 在 PendingPostPool 中的队头加入 PendingPost
  	 */
  	static void releasePendingPost(PendingPost pendingPost) {}
}
```

看了 `PendingPost` 就知道 它和 `Android Handler` 机制中的 [`Message`](https://jevishoo.github.io/2021/07/06/Android%E4%B8%AD%E7%9A%84%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/) 设计几乎一样，都使用了一个容器缓存使用过的元素，可以节省元素的创建时间。针对这种频繁创建的对象，通常可以使用这种方式提升性能、避免内存移除，没错就是享元模式。



### APT 提升效率

在**订阅阶段**是通过反射找到所有符合条件的订阅者方法，也可以通过APT的方式。APT是注解处理器的缩写，注解处理器作用于**编译阶段** ( javac )。即我们可以在编译阶段就知道所有的订阅者方法（ `@Subscribe` 修饰），并且可以在编译阶段对于不符合规定的写法提示错误，把错误留在编译期间；另一个好处是不需要到运行时通过反射获取，这样还可以提升程序运行效率。

下面显示一个编译期间提示用户方法格式错误的例子：
**@Subscribe 修饰的方法只能有一个参数。**

```dart
> Task :app:kaptDebugKotlin FAILED
/Users/aaa/workspace/EventBusDemo/app/build/tmp/kapt3/stubs/debug/com/daddyno1/eventbusdemo/MainActivity.java:21: 错误: Subscriber method must have exactly 1 parameter
    public final void onHandlerMsg(@org.jetbrains.annotations.NotNull()
                      ^
FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':app:kaptDebugKotlin'.
> A failure occurred while executing org.jetbrains.kotlin.gradle.internal.KaptExecution
   > java.lang.reflect.InvocationTargetException (no error message)
```

APT会生成辅助文件：

```java
/** This class is generated by EventBus, do not edit. */
public class MyEventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();
        
        putIndex(new SimpleSubscriberInfo(com.daddyno1.eventbusdemo.MainActivity.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onHandlerMsg", com.daddyno1.eventbusdemo.MsgEvent.class, ThreadMode.POSTING, 2, false),
            new SubscriberMethodInfo("onHandlerMsg2", com.daddyno1.eventbusdemo.MsgEvent.class, ThreadMode.POSTING, 4,
                    false),
        }));

        putIndex(new SimpleSubscriberInfo(com.daddyno1.eventbusdemo.BaseActivity.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onHandlerMsg3", com.daddyno1.eventbusdemo.MsgEvent.class, ThreadMode.POSTING, 6,
                    false),
        }));

    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```

然后使用生成的辅助文件 `MyEventBusIndex` :

```java
EventBus.builder().addIndex(MyEventBusIndex()).installDefaultEventBus()
```

这时候在 `register` 订阅者的时候，就会优先使用 `MyEventBusIndex` 中的信息去遍历订阅者的方法。
