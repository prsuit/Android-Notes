---
title: Handler消息机制
categories:
  - Android基础
tags:
  - Android基础
date: 2020-08-02 17:47:36
---

### 前言

本文是介绍Android中的Handler消息机制。

### 目录

#### 一、什么是 Handler

Handler 机制是 Android 中用于线程间通信的一套异步通信机制。官方的介绍是：

> A Handler allows you to send and process `Message` and Runnable objects associated with a thread's `MessageQueue`. Each Handler instance is associated with a single thread and that thread's message queue. When you create a new Handler it is bound to a `Looper`. It will deliver messages and runnables to that Looper's message queue and execute them on that Looper's thread.

<!--more-->

Handler 允许你发送和处理与线程的 MessageQueue 关联的 Message 和 Runnable 对象。每个 Handler 实例都与一个线程和该线程的消息队列关联。当你创建一个新的 Handler，它将绑定到 Looper。它将消息和 runnables传递到该 Looper 的消息队列，并在该 Looper 的线程上执行它们。

一句话解释 Handler 消息机制：

> Handler 通过执行其绑定线程的消息队列（MessageQueue）中不断被 Looper 循环取出的消息（Message）来完成线程间的通信。

Handler 消息机制主要包含四个类：

- **Message**：需要被传递的消息，消息分为硬件产生的消息(如按钮、触摸)和软件生成的消息；
- **MessageQueue**：负责消息的存储与管理，负责管理由 Handler 发送过来的 Message，消息队列的主要功能向消息池投递消息(`MessageQueue.enqueueMessage`)和取走消息池的消息(`MessageQueue.next`)；
- **Handler**：负责 Message 的发送及处理，主要功能向消息池发送各种消息事件(`Handler.sendMessage`)和处理相应消息事件(`Handler.handleMessage`)；
- **Looper**：负责关联线程以及消息的分发，不断循环执行(`Looper.loop`)，从 MessageQueue 获取 Message，按分发机制将消息分发给对应的 Handler 处理。

四者之间关系：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghm259akbpj30kb0iignf.jpg)

- **Looper**有一个 MessageQueue 消息队列；
- **MessageQueue**有一组待处理的 Message；
- **Message**中有一个用于处理消息的 Handler；
- **Handler**中有 Looper 和 MessageQueue。

Handler 消息机制工作流程可以理解为传送带模型：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghm4dup2wxj30rk0e8tlh.jpg)

异步消息处理线程的写法应该是这样：

```java
class LooperThread extends Thread {
  public Handler mHandler;
  
  public void run() {
    Looper.prepare();//初始化Looper，一定要写在Handler初始化之前
    mHandler = new Handler() {
      public void handleMessage(Message msg) {
        // TODO 定义消息处理逻辑
        
        //所有的事情处理完成后要退出looper，即终止Looper循环
        //这两个方法都可以
        // handler.getLooper().quit();
        // handler.getLooper().quitSafely();
      }
    };
    Looper.loop();//启动Looper循环
  }
}
```

#### 二、Looper

用于运行线程的消息循环的类。默认情况下线程没有与之关联的消息循环；在要运行循环的线程中调用 prepare() 来创建一个 Looper，然后调用 loop() 使其循环处理消息，直到循环结束为止。

##### prepare()

初始化 Looper 对象，对于无参的情况，默认调用`prepare(true)`。

```java
/** Initialize the current thread as a looper.
  * This gives you a chance to create handlers that then reference
  * this looper, before actually starting the loop. Be sure to call
  * {@link #loop()} after calling this method, and end it by calling
  * {@link #quit()}.
  */
public static void prepare() {
        prepare(true);
}
```

有参情况，参数 quitAllowed，true 表示当前 Looper 允许退出，而 false 表示当前 Looper 不允许退出。

```java
private static void prepare(boolean quitAllowed) {
    //每个线程只允许执行一次该方法，第二次执行时线程的TLS已有数据，则会抛出异常。
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //创建Looper对象，并保存到当前线程的TLS区域
    sThreadLocal.set(new Looper(quitAllowed));
}
```

Looper.prepare() 在**当前线程**使用 ThreadLocal **保存一个通过构造方法生成的Looper实例**，并在 Looper 构造方法中会创建一个 MessageQueue 对象，**保存一个MessageQueue实例**。该方法在**同一个线程只能调用一次**，所以**一个线程只会存在一个 Looper 和一个 MessageQueue** 。

看看 `sThreadLocal` 的定义：

```java
 // sThreadLocal.get() will return null unless you've called prepare().
    @UnsupportedAppUsage
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```

可以看出 `sThreadLocal` 是 ThreadLocal 类型，操作的类型都是 `Looper` 类型。

**ThreadLocal**： 线程本地存储区（Thread Local Storage，简称为TLS），提供线程局部变量，每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的TLS区域。

作用：**为每个线程提供一个独立的变量副本，以解决并发访问的冲突问题**。

本质：**ThreadLocal 的静态内部类 ThreadLocalMap 为每个 Thread 都维护了一个 Entry 类型数组 table，ThreadLocal 对象确定了一个数组下标，而这个下标就是 value 存储的对应位置。get的时候，都是从自己的变量中取值，所以不存在线程安全问题**。

Entry 类是继承自弱引用，弱引用里面放的就是 ThreadLocal 对象，Entry 的 value 存的是当前线程要存储的对象，value 作为 Entry 的成员变量。

**每个线程都持有一个 ThreadLocalMap 对象，而 threadlocal 负责访问和维护 ThreadLocalMap**。

**ThreadLocal 内存泄漏的问题，从上面分析可以发现 ThreadLocalMap 里面的 Entry 对象存储的ThreadLocal 弱引用，它并不会影响GC对于 ThreadLocal 对象的回收，但 value 直接作为 Entry 的强引用，ThreadLocal 对象可能被回收了，但 value 还在，这就造成了内存泄漏，因此在用到了 ThreadLocal 的地方，防止内存泄漏，手动调用 remove 方法。**

ThreadLocal 常用的操作方法：

- `ThreadLocal.set(T value)`：将value存储到当前线程的TLS区域，源码如下：

```java
/**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

- `ThreadLocal.get()`：获取当前线程TLS区域的数据，源码如下：

```java
 /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
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

##### 构造函数

对于 Looper 类型的构造方法如下：

```java
//构造函数私有化，外界只能通过静态方法prepare()初始化Looper
private Looper(boolean quitAllowed) {
  mQueue = new MessageQueue(quitAllowed);//创建MessageQueue对象
  mThread = Thread.currentThread();//记录当前线程.
}
```

另外，与 `prepare()` 相近功能的，还有一个 `prepareMainLooper()` 方法，该方法主要在 ActivityThread 类中使用，初始化主线程的 Looper 。

```java
/**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
    public static void prepareMainLooper() {
        prepare(false);//设置不允许退出的Looper
        synchronized (Looper.class) {
           //将当前的Looper保存为主Looper，每个线程只允许执行一次。
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

##### myLooper()

用于获取 TLS 存储的 Looper 对象，外部通过调用 Looper.myLooper() 方法获取当前线程绑定的 Looper 对象。

```java
 /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

与 `myLooper()` 相近功能的，还有一个 `getMainLooper()` 方法，获取主线程 Looper 对象。

```java
/**
     * Returns the application's main looper, which lives in the main thread of the application.
     */
    public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
```

##### loop()

loop() 方法作用是让消息队列循环起来和分发消息，需要注意的是 Looper.loop() 应该在该 Looper 所绑定的线程中执行。其核心代码如下：

```java
/**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();//获取TLS存储的Looper对象
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;//获取Looper对象中的消息队列

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        
        for (;;) { //进入loop的主循环方法
            Message msg = queue.next(); // might block 可能会阻塞
            if (msg == null) {
                // No message indicates that the message queue is quitting.
               // 没有消息，则退出循环
                return;
            }
          
            ...
              
            try {
                msg.target.dispatchMessage(msg);//用于分发Message 
              ...
            } catch (Exception exception) {
               ...
            } finally {
               ...
            }
          
            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();//将Message放入消息池
        }
    }
```

loop() 进入循环模式，不断重复下面的操作，直到没有消息时退出循环。

- **Message msg = queue.next()**，通过消息队列 MessageQueue 的 next() 方法从消息队列中取出一条消息 Message，如果此时消息队列中有 Message，那么 next 方法会立即返回该 Message，如果此时消息队列中没有 Message，那么 next 方法就会**阻塞式**地等待获取 Message ；
- **msg.target.dispatchMessage(msg)**，把 Message 分发给相应的 target ，target 值是 Handler，让 Message 所关联的 Handler 通过 dispatchMessage 方法让 Handler 处理该 Message ；
- **msg.recycleUnchecked()**，把分发后的 Message 回收到消息池，以便重复利用。

##### quit()

子线程开启的Looper，在所有的事情处理完成后要退出 Looper，即终止 Looper 循环。

```java
//退出Looper，移除所有的消息
public void quit() {
  mQueue.quit(false);
}

//安全退出Looper，只移除尚未触发的所有消息
public void quitSafely() {
  mQueue.quit(true);
}
```

Looper.quit()方法的实现最终调用的是 MessageQueue.quit() 方法：

```java
void quit(boolean safe) {
  // 当mQuitAllowed为false，表示不运行退出，强行调用quit()会抛出异常
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
          //防止多次执行退出操作
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {
                removeAllFutureMessagesLocked();//移除尚未触发的所有消息
            } else {
                removeAllMessagesLocked();//移除所有的消息
            }
            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```

注意：线程 Thread 和 Looper 是一对一绑定的，一个线程可以有多个 Handler，但只能有一个 Looper，一个MessageQueue。

#### 三、Handler

##### 无参构造

对于 Handler 的无参构造方法，默认采用当前线程 TLS 中的 Looper 对象，并且 Callback 回调方法为 null，且消息为同步处理方式。通过 ThreadLocal 获取到当前线程的 Looper 对象。

```java
public Handler() {
  this(null, false);
}

//无参构造默认会调用该构造
public Handler(@Nullable Callback callback, boolean async) {
   //匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
       //必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
        mLooper = Looper.myLooper();//从当前线程的TLS中获取Looper对象
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;//消息队列，来自Looper对象
        mCallback = callback;//回调方法
        mAsynchronous = async;//设置消息是否为异步处理方式
    }
```

##### 有参构造

Handler 类在构造方法中，可指定 Looper，Callback 回调方法以及消息的处理方式(同步或异步)，对于无 Looper 参的 Handler，默认是当前线程的 Looper 。

```java
public Handler(@Nullable Callback callback) {
  this(callback, false);
}

public Handler(@NonNull Looper looper) {
  this(looper, null, false);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback) {
  this(looper, callback, false);
}

public Handler(boolean async) {
  this(null, async);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
   mLooper = looper;
   mQueue = looper.mQueue;
   mCallback = callback;
   mAsynchronous = async;
}
```

注：在主线程中可以直接创建 Handler 对象，在子线程中需要先调用 Looper.prepare() 才能创建 Handler 对象。(因为程序启动时，ActivityThread 中的 main() 方法调用了 `Looper.prepareMainLooper()`，主线程中会始终存在一个 Looper 对象)

Callback 是 Handler 中的内部接口，Handler.Callback 是用来处理 Message 的一种手段。

```java
/**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     */
    public interface Callback {
        /**
         * @param msg A {@link android.os.Message Message} object
         * @return True if no further handling is desired
         */
        boolean handleMessage(@NonNull Message msg);
    }
```

Handler 处理 Message 的两种方法：

1.向 Hanlder 的构造函数传入一个 Handler.Callback 对象，并实现 Handler.Callback 的 handleMessage 方法；

2.无需向 Hanlder 的构造函数传入 Handler.Callback 对象，但是需要重写 Handler 本身的 handleMessage 方法。 

##### 消息分发机制

在 Looper.loop() 中，当发现有消息时，调用消息的目标 handler，执行 **dispatchMessage()** 方法来分发消息。

```java
/**
     * Handle system messages here.
     */
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
      //当Message存在回调方法，回调msg.callback.run()方法；
      handleCallback(msg);
    } else {
        if (mCallback != null) {
          //当Handler存在Callback成员变量时，回调Callback的handleMessage()；
          if (mCallback.handleMessage(msg)) {
            return;
          }
        }
      //Handler自身的回调方法handleMessage()
      handleMessage(msg);
    }
}

//执行Message的回调方法
private static void handleCallback(Message message) {
        message.callback.run();
}

 /**
     * Subclasses must implement this to receive messages.
     */
//Handler自身的回调方法
public void handleMessage(@NonNull Message msg) {
  
}
```

**分发消息流程：**

1. 当`Message`的回调方法不为空时，则回调方法`msg.callback.run()`，其中 callBack 数据类型为 Runnable ，否则进入步骤2；
2. 当`Handler`的`mCallback`成员变量不为空时，则回调方法`mCallback.handleMessage(msg)`，且回调方法要返回 true，否则进入步骤3；
3. 调用`Handler`自身的回调方法`handleMessage()`，该方法默认为空，Handler 子类通过覆写该方法来完成具体的逻辑。

##### 消息发送

发送消息调用链：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghmzljvv00j30mu0ccwer.jpg)

可以发现所有的发消息方式，最终都是调用`MessageQueue.enqueueMessage()` 方法。

**sendMessage**

```
public final boolean sendMessage(@NonNull Message msg) {
    return sendMessageDelayed(msg, 0);
}
```

**sendEmptyMessage**

```java
public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }
```

**sendEmptyMessageDelayed**

```java
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
```

**sendMessageDelayed**

```java
public final boolean sendMessageDelayed(@NonNull Message msg, long delayMillis) {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

**sendMessageAtTime**

```java
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

**sendMessageAtFrontOfQueue**

该方法通过设置消息的触发时间为0，从而使 Message 加入到消息队列的队头。

```java
public final boolean sendMessageAtFrontOfQueue(@NonNull Message msg) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, 0);
    }
```

**enqueueMessage**

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;//将Message的target绑定为当前的Handler 
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);//将消息添加到消息队列中
    }
```

**post**

```java
public final boolean post(@NonNull Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
}

//创建了一个Message对象，并将传入的Runnable对象赋值给Message的callback，然后返回该Message
private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
}
```

**postDelayed**

```java
public final boolean postDelayed(@NonNull Runnable r, long delayMillis) {
        return sendMessageDelayed(getPostMessage(r), delayMillis);
}
```

#### 四、MessageQueue

MessageQueue 是消息机制的 Java 层和 C++ 层的连接纽带，大部分核心方法都交给 native 层来处理，其中MessageQueue 类中涉及的 native 方法如下：

```java
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events)
```

##### 构造函数

```java
MessageQueue(boolean quitAllowed) {
  mQuitAllowed = quitAllowed;
  //通过native方法初始化消息队列，其中mPtr是供native代码使用
  mPtr = nativeInit();
}
```

##### enqueueMessage

添加一条消息到消息队列。

```java
 boolean enqueueMessage(Message msg, long when) {
   // 每一个普通Message必须有一个target
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {//正在退出时，回收msg，加入到消息池
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
               //p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked; //当阻塞时需要唤醒
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
               //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
               //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
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
           //消息没有退出，我们认为此时mPtr != 0
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```

MessageQueue 是按照 Message 触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。

##### next

取出下一条消息。

```java
Message next() {
        final long ptr = mPtr;
        if (ptr == 0) {//当消息循环已经退出，则直接返回
            return null;
        }

        int pendingIdleHandlerCount = -1; // 循环迭代的首次为-1
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
           //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //当遇到target为null的Message，说明是同步屏障，则查询异步消息
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                  //循环遍历找出一条异步消息，当查询到异步消息，则立刻退出循环，然后处理
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                       //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                       // 获取一条消息，并返回
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                       //设置消息的使用状态，即flags |= FLAG_IN_USE
                        msg.markInUse();
                        return msg;//成功地获取MessageQueue中的下一条即将要执行的消息
                    }
                } else {
                    // No more messages.
                   //没有消息
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
               //消息正在退出，返回null  
               if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
              //当消息队列为空，或者是消息队列的第一个消息时间大于当前时间
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                   //没有idle handlers 需要运行，则循环并等待。
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
           //只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler 去掉handler的引用

                boolean keep = false;
                try {
                    keep = idler.queueIdle();//idle时执行的方法，获取返回值
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
               //idler.queueIdle()返回false时会移除idler，只执行一次
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
           //重置idle handler个数为0，以保证不会再次重复运行
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
          //当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message.
            nextPollTimeoutMillis = 0;
        }
    }
```

`nativePollOnce` 是阻塞操作，其中`nextPollTimeoutMillis`代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直阻塞等待下去。

当处于空闲时，往往会执行`IdleHandler`中的方法。当 nativePollOnce() 返回后，next() 从`mMessages`中提取一个消息。

`nativePollOnce()`在 native 做了大量的工作，想进一步了解可查看 [Android消息机制2-Handler(native篇)](http://gityuan.com/2015/12/27/handler-message-native/#nativepollonce)。

**此处用到了Linux的pipe/epoll机制：没有消息时阻塞线程并进入休眠释放cpu资源，有消息时唤醒线程**。

##### IdleHandler

当线程将要进入堵塞，以等待更多消息时，会回调这个接口，简单点说：当 MessageQueue 中无可处理的Message 时回调。作用：**UI线程处理完所有View事务后，回调一些额外的操作，且不会堵塞主进程；**

```java
/**
     * Callback interface for discovering when a thread is going to block
     * waiting for more messages.
     */
    public static interface IdleHandler {
        /**
         * Called when the message queue has run out of messages and will now
         * wait for more.  Return true to keep your idle handler active, false
         * to have it removed.  This may be called if there are still messages
         * pending in the queue, but they are all scheduled to be dispatched
         * after the current time.
         */
        boolean queueIdle();
    }
```

接口中只有一个 queueIdle() 函数，线程进入堵塞时执行的额外操作可以写这里，返回值是 true 的话，执行完此方法后还会保留这个 IdleHandler，否则删除。

##### removeMessages

```java
void removeMessages(Handler h, int what, Object object) {
        if (h == null) {
            return;
        }
        synchronized (this) {
            Message p = mMessages;
            // Remove all messages at front.
           //从消息队列的头部开始，移除所有符合条件的消息
            while (p != null && p.target == h && p.what == what
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
            }

            // Remove all messages after front.
           //移除剩余的符合要求的消息
            while (p != null) {
                Message n = p.next;
                if (n != null) {
                    if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                        Message nn = n.next;
                        n.recycleUnchecked();
                        p.next = nn;
                        continue;
                    }
                }
                p = n;
            }
        }
    }
```

这个移除消息的方法，采用了两个 while 循环，第一个循环是从队头开始，移除符合条件的消息，第二个循环是从头部移除完连续的满足条件的消息之后，再从队列后面继续查询是否有满足条件的消息需要被移除。

##### 同步屏障机制

我们知道用 Handler 发送的 Message 后，MessageQueue 的 enqueueMessage() 按照 时间戳升序 将消息插入到队列中，而 Looper 则按照顺序，每次取出一枚Message进行分发，一个处理完到下一个。那么当**有一个紧急的Message需要优先处理怎么破？**一个 Message 分发给 Handler 后，执行了耗时操作，后面一堆本该到点执行的Message 在那里等着，这个时候你 sendMessage()，还是得排在这堆 Message 后，等他们执行完，再到你！

于是，Handler 中的 MessageQueue 加入了**同步屏障**这种机制，来实现**异步消息优先执行**的功能。

添加一个异步消息的方法很简单：

- Handler 构造方法中传入 async 参数，设置为 true，使用此 Handler 添加的 Message 都是异步的；
- 创建 Message 对象时，直接调用 setAsynchronous(true)

##### postSyncBarrier

同步消息和异步消息没太大差别，但仅限于开启同步屏障之前。postSyncBarrier 只对同步消息产生影响，对于异步消息没有任何差别。可以通过 MessageQueue 的 postSyncBarrier 函数来开启同步屏障：

```java
 public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

 //往消息队列合适的位置插入了同步屏障类型的Message (target属性为null)
    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;//初始化Msg时target没赋值，为null

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```

每一个普通 Message 必须有一个 target，对于 target 为 null 的 Message，说明是同步屏障，循环遍历找出一条异步消息，然后处理。 这个消息的价值就是用于拦截同步消息，所以并不会唤醒 Looper。

##### removeSyncBarrier

在同步屏障没移除前，只会处理异步消息，处理完所有的异步消息后，就会处于堵塞。如果想恢复处理同步消息，需要调用 removeSyncBarrier() 移除同步屏障：

```java
public void removeSyncBarrier(int token) {
        // Remove a sync barrier token from the queue.
        // If the queue is no longer stalled by a barrier then wake it.
        synchronized (this) {
            Message prev = null;
            Message p = mMessages;
          //从消息队列找到 target为空,并且token相等的Message
            while (p != null && (p.target != null || p.arg1 != token)) {
                prev = p;
                p = p.next;
            }
            if (p == null) {
                throw new IllegalStateException("The specified message queue synchronization "
                        + " barrier token has not been posted or has already been removed.");
            }
            final boolean needWake;
            if (prev != null) {
                prev.next = p.next;
                needWake = false;
            } else {
                mMessages = p.next;
                needWake = mMessages == null || mMessages.target != null;
            }
            p.recycleUnchecked();

            // If the loop is quitting then it is already awake.
            // We can assume mPtr != 0 when mQuitting is false.
            if (needWake && !mQuitting) {
                nativeWake(mPtr);
            }
        }
    }
```

#### 五、Message

##### 消息对象

每个消息用 Message 表示，Message 主要包含以下内容：

| 数据类型 | 成员变量 | 解释         |
| :------- | :------- | :----------- |
| int      | what     | 消息类别     |
| long     | when     | 消息触发时间 |
| int      | arg1     | 参数1        |
| int      | arg2     | 参数2        |
| Object   | obj      | 消息内容     |
| Handler  | target   | 消息响应方   |
| Runnable | callback | 回调方法     |

##### 消息池

静态变量`sPool`的数据类型为 Message，通过 next 成员变量，维护一个消息池；静态变量`MAX_POOL_SIZE`代表消息池的可用大小；消息池的默认大小为50。当消息池不为空时，可以直接从消息池中获取 Message 对象，而不是直接创建，提高效率。

消息池常用的操作方法是 obtain() 和 recycle()。

##### obtain

从消息池中获取消息：

```java
/**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;//从sPool中取出一个Message对象，并消息链表断开
                m.flags = 0; // clear in-use flag 清除in-use flag
                sPoolSize--;//消息池的可用大小进行减1操作
                return m;
            }
        }
        return new Message();
    }
```

obtain()，从消息池取 Message，都是把消息池表头的 Message 取走，再把表头指向 next。

##### recycle

把不再使用的消息加入消息池：

```java
public void recycle() {
        if (isInUse()) {//判断消息是否正在使用
            if (gCheckRecycle) {//Android 5.0以后的版本默认为true,之前的版本默认为false.
                throw new IllegalStateException("This message cannot be recycled because it "
                        + "is still in use.");
            }
            return;
        }
        recycleUnchecked();
    }

    // 对于不再使用的消息，加入到消息池
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
       // 将消息标示位置为FLAG_IN_USE，并清空消息所有的参数，同时将其保留在回收对象池中
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {//当消息池没有满时，将Message对象加入消息池
                next = sPool;
                sPool = this;
                sPoolSize++;//消息池的可用大小进行加1操作
            }
        }
    }
```

recycle()，将 Message 加入到消息池的过程，会把消息标示位置为 FLAG_IN_USE，并清空消息所有的参数，都是把 Message 加到链表的表头。

### 总结

**Handler 消息机制：**

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghndj328rcj30p20fatah.jpg)

- Handler 通过 sendMessage() 发送 Message 到 MessageQueue 队列；
- Looper 通过 loop()，不断提取出达到触发条件的 Message，并将 Message 交给 target 来处理；
- 经过 dispatchMessage() 后，交回给 Handler 的 handleMessage() 来进行相应地处理。
- 将 Message 加入 MessageQueue 时，往管道写入字符，可以会唤醒 loop 线程；如果 MessageQueue 中没有 Message，并处于 Idle 状态，则会执行 IdelHandler 接口中的方法，往往用于做一些清理性地工作。

**Handler：**

内部实现主要涉及到如下几个类：Thread、MessageQueue 和 Looper，它们之间的关系：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghp0auiuk8j309r06qglm.jpg)

Thread 是最基础的，Looper 和 MessageQueue 都构建在 Thread 之上，Handler 又构建在 Looper 和 MessageQueue 之上，我们通过 Handler 间接地与下面这几个相对底层一点的类打交道。

Handler 创建的时候会采用当前线程的 Looper 来构造消息循环系统，**Looper 在哪个线程创建，就跟哪个线程绑定**，并且 **Handler 处理消息是在它关联的 Looper 对应的线程中**。

**消息分发的优先级：**

1. Message 的回调方法：`message.callback.run()`，优先级最高；
2. Handler.Callback 的回调方法：`Handler.mCallback.handleMessage(msg)`，返回 true 时不会轮到步骤3，优先级仅次于1；
3. Handler 自身的默认方法：`Handler.handleMessage(msg)`，优先级最低。

**消息缓存：**

为了提供效率，提供了一个大小为50的 Message 缓存队列，减少对象不断创建与销毁的过程。

引用文章：

> [深入源码解析Handler](http://blog.csdn.net/iispring/article/details/47180325) 
> [Gityuan–消息机制Handler](http://gityuan.com/2015/12/26/handler-message-framework/)
>
> [关于Handler 的这 15 个问题，你都清楚吗？](https://mp.weixin.qq.com/s/vCnftbD3z07X79gHj30Kiw)

