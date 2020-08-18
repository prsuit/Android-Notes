---
title: HandlerThread源码分析
categories:
  - Android基础
tags:
  - Android基础
date: 2020-08-04 20:11:46
---

### 前言

本文是介绍Android中的HandlerThread。

### 目录

#### 一、HandlerThread是什么

HandlerThread 是 Android 已封装好的轻量级异步通信类。官方介绍：

> A `Thread` that has a `Looper`. The `Looper` can then be used to create `Handler`s.
>
> Note that just like with a regular `Thread`, `Thread.start()` must still be called.

一个具有 Looper 的线程。该 Looper 可以用来创建 Handler。和使用常规线程一样，`Thread.start()` 仍必须调用。

<!--more-->

**产生背景**：我们知道，耗时任务需要在子线程中进行，而线程的创建和销毁是非常消耗系统资源的，如果当任务 A 执行完了后，如果还需要执行任务 B， 那么就还需要创建一个新的子线程进行。这样性能问题就会凸显。为此，可以子线程中创建一个轮询器 Looper，当有新任务时，Looper 就开启并处理，否则就阻塞，直到下一个耗时任务的到来。因此，HandlerThread 内部封装了 Handler 和 Looper ，**可以避免多次创建和销毁线程带来的性能问题**。

**本质**：HandlerThread 是一个线程，它继承自 Thread。内部封装了 Handler 和 Looper 来进行消息的分发、循环以及处理。**通过继承Thread类**和**封装Handler类的使用**，从而使得**创建新线程和与其他线程进行通信**变得更加方便易用。

**作用**：实现多线程，在工作线程中执行任务，如耗时任务；实现异步通信、消息传递，实现工作线程和主线程(UI线程)之间的通信，即：将工作线程的执行结果传递给主线程，从而在主线程中执行相关的UI操作。

#### 二、使用步骤

```java
// 步骤1：创建HandlerThread实例对象
// 传入参数 = 线程名字，作用 = 标记该线程
   HandlerThread mHandlerThread = new HandlerThread("handlerThread");

// 步骤2：启动线程
   mHandlerThread.start();

// 步骤3：创建工作线程Handler & 复写handleMessage（）
// 作用：关联HandlerThread的Looper对象、实现消息处理操作 & 与其他线程进行通信
// 注：消息处理操作handleMessage()的执行线程 = mHandlerThread所创建的工作线程中执行
  Handler workHandler = new Handler( handlerThread.getLooper() ) {
            @Override
            public void handleMessage(Message msg) {
                ...//消息处理，通知主线程等
            }
        };

// 步骤4：使用工作线程Handler向工作线程的消息队列发送消息
// 在工作线程中，当消息循环时取出对应消息 & 在工作线程执行相关操作
  // a. 定义要发送的消息
  Message msg = Message.obtain();
  msg.what = 2; //消息的标识
  msg.obj = "B"; // 消息的存放
  // b. 通过Handler发送消息到其绑定的消息队列
  workHandler.sendMessage(msg);

// 步骤5：结束线程，即停止线程的消息循环
  mHandlerThread.quit();
```

#### 三、源码分析

##### 构造方法

```java
//传入线程名，默认优先级
public HandlerThread(String name) {
  	super(name);
    mPriority = Process.THREAD_PRIORITY_DEFAULT;
}
    
    //自定义设置优先级
    /**
     * Constructs a HandlerThread.
     * @param name
     * @param priority The priority to run the thread at. The value supplied must be from 
     * {@link android.os.Process} and not from java.lang.Thread.
     */
public HandlerThread(String name, int priority) {
    super(name);
    mPriority = priority;
}
```

- `HandlerThread`类继承自`Thread`类；
- 创建`HandlerThread`类对象 = 创建`Thread`类对象 + 设置线程优先级 = **新开1个工作线程 + 设置线程优先级**。

##### run

启动线程，`mHandlerThread.start()`最终会回调 HandlerThread 的 run() 方法。

```java
@Override
public void run() {
  mTid = Process.myTid();
  Looper.prepare(); //为当前线程创建Looper对象
  synchronized (this) {//同步代码块
  	mLooper = Looper.myLooper();//获取Looper对象
    //阻塞--等待机制
    //发出通知，当前线程已经创建mLooper对象成功，这里主要是通知getLooper()中的等待锁wait()，结束阻塞等待
    notifyAll();
    //此处使用持有锁机制 + notifyAll() 是为了保证后面获得Looper对象前就已创建好Looper对象
  }
  Process.setThreadPriority(mPriority);//设置当前线程的优先级
  onLooperPrepared();//在线程循环消息之前做一些准备工作
  Looper.loop(); //开启循环，Handler从消息队列中处理消息
  mTid = -1;
}
```

- 为当前工作线程（即步骤1创建的线程）创建1个`Looper`对象 & `MessageQueue`对象；

- 通过持有锁机制来获得当前线程的`Looper`对象；

- 发出通知：当前线程已经创建 mLooper 对象成功；

- 工作线程进行消息循环，即不断从 MessageQueue 中取消息 & 派发消息。

##### onLooperPrepared

重写该方法，可在 Looper 创建完成之后，开始循环消息之前可做一些准备工作。

```java
 /**
     * Call back method that can be explicitly overridden if needed to execute some
     * setup before Looper loops.
     * 重写该方法，可在循环消息之前可做一些准备工作，在Loop.loop()方法之前调用
     */
    protected void onLooperPrepared() {
    }
```

##### getLooper

获得当前 HandlerThread 线程中的 Looper 对象。

```java
/**
     * This method returns the Looper associated with this thread. If this thread not been started
     * or for any reason isAlive() returns false, this method will return null. If this thread
     * has been started, this method will block until the looper has been initialized.  
     * @return The looper.
     */
    public Looper getLooper() {
       // 若线程不是存活的，则直接返回null
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
      //同步代码块：如果线程已经启动，但是Looper还未创建的话，就阻塞等待，直到run()中的Looper对象创建成功
        synchronized (this) {
            //阻塞-等待机制，确保已经创建了mLooper对象
            //开始循环等待，调用wait()去阻塞线程，
            //当run()中的notifyAll()调用之后，通知当前线程的wait方法结束等待，跳出循环。
            while (isAlive() && mLooper == null) {
                try {
                    wait();//阻塞线程，等待
                } catch (InterruptedException e) {
                }
            }
        }
        // 等待唤醒后，结束等待，跳出循环，返回在run()中创建的mLooper对象
        return mLooper;
    }
```

- 在获得`HandlerThread`工作线程的`Looper`对象时存在一个同步的问题：只有当线程创建成功 & 其对应的`Looper`对象也创建成功后才能获得`Looper`的值，才能将创建的`Handler` 与 工作线程的`Looper`对象绑定，从而将`Handler`绑定工作线程。

- 解决方案：即保证同步的解决方案 = 同步锁、`wait（）` 和 `notifyAll（）`，即 在`run()`中成功创建`Looper`对象后，立即调用`notifyAll（）`通知 `getLooper()`中的`wait（）`结束等待 & 返回`run()`中成功创建的`Looper`对象，使得`Handler`与该`Looper`对象绑定。

##### quit/quitSafely

在 HandlerThread 不使用的时候，需要调用退出方法`quit()/quitSafely()`，结束线程，即停止线程的消息循环。

```java
 public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

 public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }
```

### 总结

- HandlerThread 本质是一个线程类，继承自 Thread；
- HandlerThread 有自己的内部 Looper 对象，可以进行 Looper 循环；
- 通过获取 HandlerThread 的 Looper 对象传递给 Handler，可以在 `handlerMessage()`中执行异步任务；
- 优点是减少了对性能的消耗，缺点是不能同时进行多任务的处理，需要等待处理，效率较低；
- 与线程池注重并发不同，HandlerThread 是一个串行队列，HandlerThread 背后已只有一个线程；
- 在 HandlerThread 不使用的时候，需要调用退出方法`quit()/quitSafely()`。

[Demo](https://github.com/prsuit/Android-Learn-Sample)

引用文章：

> [Android多线程：这是一份详细的HandlerThread源码分析攻略](https://www.jianshu.com/p/4a8dc2f50ae6)
>
> [Android消息机制之HandlerThread](https://blog.csdn.net/hust_twj/article/details/87884318?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

