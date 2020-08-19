---
title: IntentService源码分析
categories:
  - Android基础
tags:
  - Android基础
date: 2020-08-06 21:14:48
---

### 前言

本文是介绍Android中的IntentService。

### 目录

#### 一、IntentService是什么

IntentService 是 Android 里的一个封装的抽象类，继承自 Service。官方介绍：

> IntentService is a base class for `Service`s that handle asynchronous requests (expressed as `Intent`s) on demand. Clients send requests through `Context.startService(Intent)` calls; the service is started as needed, handles each Intent in turn using a worker thread, and stops itself when it runs out of work.

<!--more-->

IntentService 是处理异步请求服务的基类。客户端通过调用 Context.startService(Intent)发送请求，服务会使用一个工作线程依次处理每个意图，并在用完工作时自动停止。

**产生背景**：如果想要在 Service 中做些耗时的操作，需要开启一个子线程然后内部执行耗时的操作，在执行完毕后如果需要自动停止服务需要在子线程的 run 方法中调用 stopSelf() 来停止服务，IntentService 类可以很方便的解决了自己开启线程和手动停止服务的问题，适合需要在工作线程处理 UI 无关任务的场景。

**本质**：可以看做是 Service 和 HandlerThread 的结合体，封装了一个 HandlerThread 和 Handler 的异步框架，在完成了任务之后会自动停止。

**作用**：处理异步请求和实现多线程。

#### 二、使用步骤

IntentService 是一个抽象类内部有一个抽象的方法 `onHandleIntent()`，继承至 Service 类。所以使用它需要继承它实现抽象方法。

```java
  public class MyIntentService extends IntentService {
  private static final String TAG = "MyIntentService";
    
  //构造函数，传入的字符串就是worker thread的名字  
  public MyIntentService() {
      super("MyIntentService");
  }

  @Override
  protected void onHandleIntent(Intent intent) {
      //在这里通过intent携带的数据，开进行任务的操作。
      Log.d(TAG, "onHandleIntent: " + Thread.currentThread().getName());
  }

  @Override
  public void onDestroy() {
      super.onDestroy();
      Log.d(TAG, "onDestroy: ");
    }
}
```

然后调用 StartService() 启动异步后台服务类 IntentService。

```java
  startService.setOnClickListener(new View.OnClickListener() {
          @Override
          public void onClick(View v) {
              Intent intent = new Intent(MainActivity.this, MyIntentService.class);
              startService(intent);
          }
      });
```

#### 三、源码分析

#####  onCreate

方法会开启一个新的子线程 HandlerThread ( HandlerThread 内部封装了 Looper )，然后创建一个 ServiceHandler 与子线程中的 Looper 对象绑定。

```java
@Override
public void onCreate() {
    super.onCreate();
    
    // 1. 通过实例化andlerThread新建线程 & 启动；故 使用IntentService时，不需额外新建线程
    // HandlerThread继承自Thread，内部封装了 Looper
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();
  
    // 2. 获得工作线程的 Looper & 维护自己的工作队列
    mServiceLooper = thread.getLooper();

    // 3. 新建mServiceHandler & 绑定上述获得Looper
    // 新建的Handler 属于工作线程 ->>分析1
    mServiceHandler = new ServiceHandler(mServiceLooper); 
}

   /** 
     * 分析1：ServiceHandler源码分析
     **/ 
     private final class ServiceHandler extends Handler {

         // 构造函数
         public ServiceHandler(Looper looper) {
         super(looper);
       }

        // IntentService的handleMessage（）把接收的消息交给onHandleIntent()处理
        @Override
         public void handleMessage(Message msg) {
  
          // onHandleIntent 方法在工作线程中执行
          // onHandleIntent() = 抽象方法，使用时需重写 ->>分析2
          onHandleIntent((Intent)msg.obj);
          // 执行完调用 stopSelf(startId) 结束服务
          stopSelf(msg.arg1);

    }
}
```

##### onHandleIntent

重写此方法进行耗时任务操作处理，该方法在工作线程上调用并请求处理，一次只处理一个意图，处理完所有意图请求后，会调用 `stopSelf(satrtId)` 停止 IntentService。

```java
/** 
     * 分析2： onHandleIntent()源码分析
     * onHandleIntent() = 抽象方法，使用时需重写
     **/ 
@WorkerThread
protected abstract void onHandleIntent(@Nullable Intent intent);
```

##### stopSelf

根据传入的 startId 来判断是否停止 IntentService。

```java
public final void stopSelf() {
        stopSelf(-1);
}

public final void stopSelf(int startId) {
        if (mActivityManager == null) {
            return;
        }
        try {
            mActivityManager.stopServiceToken(
                    new ComponentName(this, mClassName), mToken, startId);
        } catch (RemoteException ex) {
        }
}
```

在stopServiceToken() ->> mServices.stopServiceTokenLocked() 方法中会进行判断：

- startId < 0：stopSelf() 方法会直接停止 Service；
- startId > 0 && startId != ServiceRecord.lastStartId：不会停止 Service，代码流程直接返回；
- startId > 0 && startId == ServiceRecord.lastStartId：直接停止 Service。

##### onStartCommand

将 Intent 传递给 ServiceHandler 并依次插入到工作队列中，在方法中调用`onStart()`构建一个 Message 对象，并且将传递进来的 Intent 封装在 Message，通过 mServiceHandler 发送到消息队列中。经过 Looper 循环将消息分发到 ServiceHandler 的 handleMessage 中处理，从而传递给 `onHandleIntent()` 方法。

```cpp
/** 
  * onStartCommand（）源码分析
  * onHandleIntent() = 抽象方法，使用时需重写
  **/ 
  public int onStartCommand(Intent intent, int flags, int startId) {

    // 调用onStart（）->>分析1
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}

/** 
  * 分析1：onStart(intent, startId)
  **/ 
  public void onStart(Intent intent, int startId) {

    // 1. 获得ServiceHandler消息的引用
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;

    // 2. 把 Intent参数 包装到 message 的 obj 发送消息中，
    //这里的Intent  = 启动服务时startService(Intent) 里传入的 Intent
    msg.obj = intent;

    // 3. 发送消息，即 添加到消息队列里
    mServiceHandler.sendMessage(msg);
}
```

##### onDestroy

IntentService 销毁时，停止 Looper 并把消息队列中的所有消息给移除。

```java
 @Override
   public void onDestroy() {
       //将消息队列中的所有消息给移除，包括处理中的和未处理的
       mServiceLooper.quit();
   }
```

注意：

**1.为什么不建议通过 bindService() 启动 IntentService？**

IntentService 源码中的 onBind() 默认返回 null，bindService() 启动服务生命周期：onCreate() ->> onBind() ->> onunbind()->> onDestory()，并不会调用`onStart()` 或 `onStartcommand()`，**故不会将消息发送到消息队列，那么onHandleIntent()将不会回调，即无法实现多线程的操作**。

**2.为什么多次启动 IntentService 会顺序执行事件，停止服务后，后续的事件得不到执行？**

由于`onCreate()`只会调用一次 = 只会创建1个工作线程；

当多次调用 `startService(Intent)`时（即 `onStartCommand（）`也会调用多次），其实不会创建新的工作线程，只是把消息加入消息队列中等待执行。而如果服务停止，会清除消息队列中的消息，后续的事件得不到执行。

一个请求处理完毕，如果此时没有新的请求，那么 IntentService 会进行销毁；一个请求还在处理过程中，如果有新的请求，那么 IntentService 会在本次请求处理完成后接着去处理新的请求，此时不会销毁。待所有请求处理完毕后，再进行销毁。是通过调用 stopSelf(satrtId) 方法 根据传入的 startId 来判断是否停止 IntentService。

##### 3.在源码handleMessage方法中为什么执行完onHandleIntent方法会去调用带参数的stopSelf(satrtId)？

因为 `stopSelf()` 的执行会立刻将服务停止掉，而带参数的 `stopSelf(int startId) `会在所有任务执行完毕后将服务给停止。通常情况下调用 `stopSelf(int satrtId)` 方法会去判断最近执行意图的 startId 是否和传入的 startId 相等，如果相等就立刻执行停止服务的操作。

**4.启动 IntentService 为什么不需要新建线程？**
IntentService 内部的 HandlerThread 继承自 Thread，内部封装了 Looper，在这里新建线程并启动，所以启动 IntentService 不需要新建线程。

**5.onHandleIntent() 方法是在工作线程中执行**

### 总结

IntentService 是继承自 Service 并处理异步请求的一个类，在 IntentService 内有一个工作线程来处理耗时操作。

当任务执行完后，IntentService 会自动停止，不需要我们去手动结束。

如果启动 IntentService 多次，那么每一个耗时操作会以工作队列的方式在 IntentService 的 onHandleIntent 回调方法中执行，依次去执行，使用串行的方式，执行完自动结束。

[Demo](https://github.com/prsuit/Android-Learn-Sample)

引用文章：

> [从源码角度理解HandlerThread和IntentService](https://blog.csdn.net/xlh1191860939/article/details/107225817)
>
> [Android多线程：这是一份全面 & 详细的IntentService源码分析指南](https://www.jianshu.com/p/8a3c44a9173a)
>
> [Android IntentService使用介绍以及原理分析](https://www.jianshu.com/p/8c4181049564)

