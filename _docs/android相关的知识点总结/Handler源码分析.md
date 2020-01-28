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

**1.Handler.java**

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

**<2>** **sendMessage方法**





