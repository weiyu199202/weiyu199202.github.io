---
title: Handler源码分析
category: android相关的知识点总结
order: 1
---

#### 第一部分：基础定义

**作用**：子线程与主线程通过Handler来进行通信。子线程可以通过Handler来通知主线程进行UI更新。（准确来讲指的是线程间的通信）

**过程**：handler通过 sendMessage 方法发送信息到消息队列 MessageQueue，Looper不断轮询MessageQueue里面的消息Message，一旦有未延时的消息，就回调用handler的dispatchMessage方法然后返回给handleMessage方法使用。

原理图如下:

![](\images\handler原理图.png)

核心类和作用：

| 主要类           | 作用                                                  |
| :--------------- | ----------------------------------------------------- |
| **Handler**      | 用来发送和处理消息。android里面一般位于主线程。       |
| **Message**      | 用来承载handler发送的内容，存放在**MessageQueue中。** |
| **MessageQueue** | 用来装载消息的容器（或者数据结构）。                  |
| **Looper**       | 不断轮询**MessageQueue**里面的**Message**             |
|                  |                                                       |

#### **第二部分：源码解析：**

##### **1.Handler.java**

**<1>构造方法：**

Handler 的构造方法有很多：如下

```
public Handler() {
    this(null, false);
}

public Handler(Callback callback) {
    this(callback, false);
}

public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}

public Handler(boolean async) {
    this(null, async);
}

public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        ......//省略代码，因为FIND_POTENTIAL_LEAKS 恒为false
   }
   
   mLooper = Looper.myLooper();//创建了Looper对象
  if (mLooper == null) {
    throw new RuntimeException(
        "Can't create handler inside thread that has not called Looper.prepare()");
   }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
    

```

**由于系统源码**FIND_POTENTIAL_LEAKS 是 false  所以代码可以忽略

```
private static final boolean FIND_POTENTIAL_LEAKS = false;
```

通过观察发现：构造函数最后都直接或者间接的调用了最后两个构造函数，也就是：

- Handler(Callback callback, boolean async) 
- Handler(Looper looper, Callback callback, boolean async)

并且两个构造函数的区别只是后者比前者多传递了一个Looper对象。前者是通过**Looper.myLooper()**方法自动创建了一个Looper对象。到此我们得到：

- 通过 looper.mQueue我们知道：MessageQueue是由Looper得到的
- 创建Handler 时候，Looper可以传递 ，也可以不传递。不传递的时候，系统会自动创建一个Looper

##### **<2>** **sendMessage方法**

![](\images\sendMessage方法.png)

如上图，发送消息的方法有很多，但是最后都会调用enqueueMessage方法，源码如下：

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;//this就是Handler本身
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

从中我们知道

- （1）Message是通过queue.enqueueMessage插入到消息队列
- （2）Message 有个target 属性是Handler-----很重要

自此，关于消息传递的Handler部分基本讲完了，但是还有一部分要将的是一个特殊函数：

```
dispatchMessage(Message msg)
```

这个函数源码如下：

```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

```

由代码我们可得到下面信息

- Message 会有一个自己的callback 单独的回调，调用callback 自身的run方法，必要的时候可以使用。会优先调用。
- 如果Message 自己的callback 为空，那么就会调用传入的callback.handleMessage(msg) 方法。
- 如果callback 为空，最后才会调用Handler自身的handleMessage方法。三个方法是互斥的关系。

##### **2.Message.java**

**(1)看构造方法：**

![](\images\Message.png)

**构造方法也很多，但是只需要看一个参数最全的就可以：**

```
private static final Object sPoolSync = new Object();
private static Message sPool;
private static int sPoolSize = 0;

private static final int MAX_POOL_SIZE = 50;

private static boolean gCheckRecycle = true;

/**
 * Return a new Message instance from the global pool. Allows us to
 * avoid allocating new objects in many cases.
 */
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}

//只需要看这个构造方法
public static Message obtain(Handler h, int what,
        int arg1, int arg2, Object obj) {
    Message m = obtain();
    m.target = h;
    m.what = what;
    m.arg1 = arg1;
    m.arg2 = arg2;
    m.obj = obj;
    return m;
}

```

得到的信息：

- 1.Message 维护着一个对象池 sPoolSync   对象池的大小最大为MAX_POOL_SIZE = 50。
- 2.Message 有一个属性target 是Handler 类型的

##### **3.MessageQueue.java**

**(1)看构造方法：**

```
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {//必须要有handler
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            //如果消息到了就把needWake唤醒状态设置为true
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);//唤醒操作，
        }
    }
    return true;
}

```

我们一直称MessageQueue为消息队列，实际上，看代码，里面使用**链表**实现的。

##### **4.Looper.java**

**(1)看构造方法：**

```
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

```

可以看到Looper是存在一个sThreadLocal的对象里面，并且每个线程只会对应一个Looper对象。接下来是获取Looper的方法，自己可以推测出了：

```
public static Looper myLooper() {
    return sThreadLocal.get();
}
```

这边要看下比较特殊的，MainLooper是系统自动创建的，所以我们再主线程中使用Handler 时候，并不需要去主动创建自己的Looper。

```

public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}


public static Looper getMainLooper() {
    synchronized (Looper.class) {
        return sMainLooper;
    }
}

```

然后看 loop()的最核心的方法，轮询的过程。

```
public static void loop() {
    final Looper me = myLooper();//得到Looper对象
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;//得到MessageQueue 对象

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {//for死循环轮询，在没消息来的时候这里面会阻塞，但是一旦有消息，系统就会唤醒
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        final long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;

        final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        final long end;
        try {
            //这边msg.target就是前面说到的Handler
            //,也就是Handler发送的消息为什么会被自身接收到的原因
            msg.target.dispatchMessage(msg);
            end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
      //省略代码........
    }
}

```

在这边要说下：

1.loop()会有一个for死循环轮询，在没消息来的时候这里面会阻塞，但是一旦有消息，系统就会唤醒。

\2. 为什么死循环了，系统不会出现“长时间未响应”或者ANR的错误，这是由于这两个错误都是在生命周期中函数产生的，android事件机制导致的。生命周期都是由Handler机制驱动的，那么我们先从入口ActivityThread 类开始看：首先 ActivityThread 并不是一个 Thread，就只是一个 final 类而已。我们常说的主线程就是从这个类的 main 方法开始，main 方法很简短，一眼就能看全（如上），我们看到里面有 Looper 了，那么接下来就找找 ActivityThread 对应的 Handler 啊，就是内部类 H，其继承 Handler，贴出 handleMessage 的小部分：

![](\images\handleMessage.png)

如果你了解下linux的epoll你就知道为什么不会被卡住了，先说结论：阻塞是有的，但是不会卡住 

主要原因有2个

1.epoll模型 当没有消息的时候会epoll.wait，等待句柄写的时候再唤醒，这个时候其实是阻塞的。

2.所有的ui操作都通过handler来发消息操作。 

比如屏幕刷新16ms一个消息，你的各种点击事件，所以就会有句柄写操作，唤醒上文的wait操作，所以不会被卡死了。

这里涉及线程，先说说说进程/线程： 

进程：每个app运行时前首先创建一个进程，该进程是由Zygote fork出来的，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。 

线程：线程对应用来说非常常见，比如每次new Thread().start都会创建一个新的线程。该线程与App所在进程之间资源共享，从Linux角度来说进程与线程除了是否共享资源外，并没有本质的区别，都是一个task_struct结构体，在CPU看来进程或线程无非就是一段可执行的代码，CPU采用CFS调度算法，保证每个task都尽可能公平的享有CPU时间片。

[参考链接]: https://www.zhihu.com/question/34652589/answer/90344494

**(1) Android中为什么主线程不会因为Looper.loop()里的死循环卡死？**

这里涉及线程，先说说说进程/线程，**进程：**每个app运行时前首先创建一个进程，该进程是由Zygote fork出来的，用于承载App上运行的各种Activity/Service等组件。进程对于上层应用来说是完全透明的，这也是google有意为之，让App程序都是运行在Android Runtime。大多数情况一个App就运行在一个进程中，除非在AndroidManifest.xml中配置Android:process属性，或通过native代码fork进程。

**线程：**线程对应用来说非常常见，比如每次new Thread().start都会创建一个新的线程。该线程与App所在进程之间资源共享，从Linux角度来说进程与线程除了是否共享资源外，并没有本质的区别，都是一个task_struct结构体**，在CPU看来进程或线程无非就是一段可执行的代码，CPU采用CFS调度算法，保证每个task都尽可能公平的享有CPU时间片**。

有了这么准备，再说说死循环问题：

对于线程既然是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，那么如何保证能一直存活呢？**简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出，**例如，binder线程也是采用死循环的方法，通过循环方式不同与Binder驱动进行读写操作，当然并非简单地死循环，无消息时会休眠。但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？通过创建新线程的方式。

真正会卡死主线程的操作是在回调方法onCreate/onStart/onResume等操作时间过长，会导致掉帧，甚至发生ANR，looper.loop本身不会导致应用卡死。

##### **5.ThreadLocal.java**

在前面的Looper俩面提到ThreadLocal 现在来讲一下：

```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

1.set方法：

```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

由此可以知道里面维护一个ThreadLocalMap   并且和当前线程绑定了，说明looper和当前线程绑定

2.get方法

```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

```

通过getEntry方法知道里面维护了一个数组

```
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

具体参照 https://www.cnblogs.com/ldq2016/p/9041856.html



#### **第三部分：试题：**

1. **Handler 消息机制是什么？**

   线程间通过handler通信。主要：子线程用主线程的handler对象发送Message消息通知主线程更新UI。

2. **为什么不能在子线程中访问UI?**

   （1）UI操作没加锁，不安全。如果在多线程中并发访问可能会导致UI控件处于不可预期的状态。

   （2）事实可以操作，但是必须在 ViewRootImpl.checkThread()方法之前调用。不然checkThread()会检查到当前线程不是主线程，抛出异常。

3. **那为什么不对UI控件的访问加上锁机制呢？**

    缺点有两个：

   （1）首先加上锁机制会让UI访问的逻辑变得复杂。

   （2）其次锁机制会降低UI访问的效率。

4. **在子线程中创建Handler报错是为什么?**

   没有加上Looper.prepare();

5. **如何在子线程创建Looper？** 

   Looper.prepare()

6. **为什么通过Handler能实现线程的切换**

   Looper就是一个消息循环，它的内部维护着一个消息队列。一旦它开始消息循环，它就一直在那个线程内执行循环，它的消息是在哪个线程加入的，不影响这个循环。我们可以看到，MessageQueue的enqueueMessage()方法是做了线程同步的

7. **Handler为什么不能实现进程间通讯**

   在工作的线程与主线程共享内存空间，即被实例的mHandler在共享空间的内存堆上，正在工作的线程与主线程都可以操作mHandler，正在工作的线程通过mHandler向其成员变量MessageQueue中添加新的Message,主线程一直处于Loop()中，当主线成接收到新的消息时通过一定的规则传递给handlerMessage(),而进程之间内存空间是不共享的，所以Handler不能实现进程之间的通讯。

8. **Handler.post的逻辑在哪个线程执行的，是由Looper所在线程还是Handler所在线程决定的？**

    由Looper所在线程决定的，最终逻辑是在Looper.loop()方法中，从MsgQueue中拿出msg，并且执行其逻辑，这是在Looper中执行的，因此有Looper所在线程决定。

9. **Looper和Handler一定要处于一个线程吗？子线程中可以用MainLooper去创建Handler吗？**

   （1）不一定   （2）可以

10. **Handler的post/send()的原理是什么**

    (1)msg的callback不为空，调用handleCallback方法（message.callback.run()）----post

    (2)mCallback不为空，调用mCallback.handleMessage(msg)----

    (3)最后如果其他都为空，执行Handler自身的 handleMessage(msg) 

11. **Handler的post方法发送的是同步消息吗？可以发送异步消息吗？**

    用户层面发送的都是同步消息;

    消息队列循环执行，不一定是完全按照时间串行执行的，是可以有异步消息的。

12. **Handler的post()和postDelayed()方法的异同？**

    post()就是调用 postDelayed()

13. **Handler的postDelayed的底层机制**

    (1)postDelay()一个10秒钟的Runnable A、消息进队，MessageQueue调用nativePollOnce()阻塞，Looper阻塞；

    (2)紧接着post()一个Runnable B、消息进队，判断现在A时间还没到、正在阻塞，把B插入消息队列的头部（A的前面），然后调用nativeWake()方法唤醒线程；

    \(3)MessageQueue.next()方法被唤醒后，重新开始读取消息链表，第一个消息B无延时，直接返回给Looper；

    (4) Looper处理完这个消息再次调用next()方法，MessageQueue继续读取消息链表，第二个消息A还没到时间，计算一下剩余时间（假如还剩9秒）继续调用nativePollOnce()阻塞；

    (5) 直到阻塞时间到或者下一次有Message进队；

14. **MessageQueue.next()会因为发现了延迟消息，而进行阻塞。那么为什么后面加入的非延迟消息没有被阻塞呢？**

    底层会唤醒

15. **Handler的dispatchMessage()分发消息的处理流程？**

    三层机制  

16. **Handler为什么要有Callback的构造方法？**

    不需要派生Handler   直接在操作内中执行逻辑

17. **Handler构造方法中通过Looper.myLooper();是如何获取到当前线程的Looper的？**

    myLooper()内部使用ThreadLocal实现，因此能够获取各个线程自己的Looper

18. **主线程如何向子线程发送消息？**

    一样的  在子线程new 一个Handler

19. **MessageQueue是什么？**

    存放Message的集合(单向链表)

20. **MessageQueue的主要两个操作是什么?有什么用？**

     enqueueMessage   添加和取出message

21. **.MessageQueue中底层是采用的队列？**

    采用单链表的数据结构来维护消息队列，而不是采用队列

22. **MessageQueue的enqueueMessage()方法的原理，如何进行线程同步的？**

    （1）就是单链表的插入操作

    （2） 如果消息队列被阻塞回调用nativeWake去唤醒。

    （3）用synchronized代码块去进行同步。

23. **Looper.loop()是如何阻塞的？MessageQueue.next()是如何阻塞的？** 

    （1）MessageQueue.next()阻塞引起的

    （2）MessageQueue无消息或者MessageQueue有延迟消息就会阻塞

    通过native方法：nativePollOnce()进行精准时间的阻塞。

24. **Looper是什么？** 

    message轮询器

25. **如何开启消息循环？**

     Looper.loop();

26. **Looper的构造内部会创建消息队列。**

    会

27. **Looper的两个退出方法？**

    当我们调用Looper的quitSafely方法时，实际上执行了MessageQueue中的removeAllFutureMessagesLocked方法，通过名字就可以看出，该方法只会清空MessageQueue消息池中所有的延迟消息，并将消息池中所有的非延迟消息派发出去让Handler去处理，quitSafely相比于quit方法安全之处在于清空消息之前会派发所有的非延迟消息。

28. **子线程中创建了Looper，在使用完毕后，终止消息循环的方法？**

    quitSafely

    quit

29. **Looper.loop()的源码流程?**

    （1）获取到Looper和消息队列

    （2）for无限循环，阻塞于消息队列的next方法

    （3）取出消息后调用msg.target.dispatchMessage(msg)进行消息分发

30. **Looper.loop()在什么情况下会退出？**

    （1）next方法返回的msg == null

    （2）线程意外终止

31. **MessageQueue的next方法什么时候会返回null？**

    执行 quitSafely   quit

32. **Looper.quit/quitSafely的本质是什么？**

    让消息队列的next()返回null，依次来退出Looper.loop()

33. **Looper.loop()方法执行时，如果内部的myLooper()获取不到Looper会出现什么结果?**

    抛异常

34. **主线程是如何准备消息循环的？**

    （主线程的Looper如何进行初始化的）

    ActivityThread中main方法中直接调用 Looper.prepareMainLooper();初始化

35. **ActivityThread中的Handler 的作用？**

    常用生命周期正是在这个Handler里面执行的，可见这个Looper承载着整个应用生命周期。

36. **如何获取主线程的MainLooper？**

    Looper.getMainLooper()

37. **Android如何保证一个线程最多只能有一个Looper？如何保证只有一个MessageQueue？**

    ThreadLocal 保证一个线程最多只能对应一个 Looper。Looper自带MessageQueue.，从而保证MessageQueue的唯一

38. **Handler消息机制中，一个looper是如何区分多个Handler的？**

    （1）Looper里面其实还有一个属性叫做target,这个target属性就是handler。

    一个handler发送的消息只会被自己接收。

    （2）发送消息除了利用Handler之外还有Message，Message一般利用Obtain获取，其实obtain中还可以传递参数，可以接收Message，还可以接收Handler，message有多个属性，常用的有what，arg1,arg2,data等，（区分message）

39. **ThreadLocal是什么？**

    ThreadLocal是一个本地线程副本变量工具类。主要用于将私有线程和该线程存放的副本对象做一个映射，各个线程之间的变量互不干扰，在高并发场景下，可以实现无状态的调用，特别适用于各个线程依赖不通的变量值完成操作的场景。

40. **ThreadLocal的使用**

    同一个ThreadLocal调用set(xxx)和get()

41. **ThreadLocal的原理**

    thread.threadLocals就是当前线程thread中的ThreadLocalMap

    ThreadLocalMap中有一个table数组，元素是Entry。根据ThreadLocal(需要转换获取到Hash Key)能get到对应的Enrty。

    Entry中key为ThreadLocal, value就是存储的数值。

42. **如何获取到当前线程**

    Thread.currentThread()就是当前线程。

43. **如何在ThreadLocalMap中，ThreadLocal如何作为键值对中的key？**

    通过ThreadLocal计算出Hash key，通过这个哈希值来进行存储和读取的。



