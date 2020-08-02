---
title: BroadcastReceiver必备基础
date: 2020-07-27 20:43:26
categories: [Android基础]
tags: [Android基础]
---

### 前言

本文是介绍Android的四大组件之一的BroadcastReceiver。

### 目录

#### 一、什么是BroadcastReceiver

广播是一种广泛运用的在应用程序之间传输信息的机制，**BroadcastReceiver** 主要用来监听系统或者应用发出的广播信息，然后根据广播信息作为相应的逻辑处理，也可以用来传输少量、频率低的数据。广播机制是一个典型的发布-订阅模式，就是观察者模式。

<!--more-->

#### 二、广播的种类

##### 普通广播(Normal Broadcast)

普通广播是完全异步的，可以在同一时刻（逻辑上）被所有接收者接收到，所有满足条件的 BroadcastReceiver 都会随机地执行其 onReceive() 方法。同级别接收是先后是随机的；级别低的收到广播；消息传递的效率比较高，并且无法中断广播的传播。

##### 有序广播(Ordered Broadcast)

有序广播是一种同步执行的广播。在有序广播发出之后同一时刻只会有一个广播接收器能收到这条广播消息，当这个广播接收器中的逻辑执行完毕后，广播才会继续传播。

所有的广播接收器按照优先级顺序依次执行，优先级高的先执行，低的后执行。广播接收器的优先级通过清单文件中 receiver 标签下 `intent-filter ` 中的 `android:priority` 属性来设置，数值越大优先级越高，数值越小优先级越低。

在广播传递的过程中可对广播进行中断操作，通过在 onReceiver() 方法中调用 abortBroadcast() 方法实现，这样后面的广播接收器就无法收到广播消息了。

##### 本地广播(Local Broadcast)

为了解决广播的安全性问题，Android引入了本地广播机制，该机制能够在应用内部进行传递，并且广播接收器也只能接收来自本应用发出的广播，安全性高和效率高。本地广播主要是一个 LocalBroadcastManager 对广播进行管理，并提供发送跟注册广播接收器的方法。

##### 粘性广播(Sticky Broadcast)

指广播在发出后，还会保存在 AMS 中，在广播发出之后注册的广播接收器也能收到之前发送的该广播，该广播在 Android 5.0 被废弃。

#### 三、广播的使用

#####  创建BroadcastReceiver

需要继承 BroadcastReceiver 基类，并重写 onReceive() 方法来接收消息。

```java
// 继承BroadcastReceivre基类
public class CustomBroadcastReceiver extends BroadcastReceiver {
  // 复写onReceive()方法
  // 接收到广播后，则自动调用该方法
  @Override
  public void onReceive(Context context, Intent intent) {
   //写入接收广播后的操作
    }
}
```

##### 注册BroadcastReceiver

- **静态注册**

  在 AndroidManifest.xml 里通过 `<receiver>` 标签声明，并在标签内用 `<intent-filter>` 标签设置过滤器。

  ```java
  <application>
   <receiver 
      //是否可以被系统实例化
      android:enabled=["true" | "false"]
     //此broadcastReceiver能否接收其他App的发出的广播
     //默认值是由receiver中有无intent-filter决定的：如果有intent-filter，默认值为true，否则为false
      android:exported=["true" | "false"]
      android:icon="drawable resource"
      android:label="string resource"
     //继承BroadcastReceiver子类的类名
      android:name=".CustomBroadcastReceiver"
   //具有相应权限的广播发送者发送的广播才能被此BroadcastReceiver所接收；
      android:permission="string"
  //BroadcastReceiver运行所处的进程
  //默认为app的进程，可以指定独立的进程
  //注：Android四大基本组件都可以通过此属性指定自己的独立进程
      android:process="string" >
  //用于指定此广播接收器将接收的广播类型
    <intent-filter>
      <action android:name="com.sample.custom.actionName" />
    </intent-filter>
   </receiver>
  </application>
  ```

- **动态注册**

  在代码中调用`Context.registerReceiver()`方法注册，`unregisterReceiver()` 方法取消注册。

  ```java
  // 选择在Activity生命周期方法中的onResume()中注册
  @Override
    protected void onResume(){
        super.onResume();
      // 1. 实例化BroadcastReceiver子类 &  IntentFilter
       mBroadcastReceiver mBroadcastReceiver = new CustomBroadcastReceiver();
       IntentFilter intentFilter = new IntentFilter();
      // 2. 设置接收广播的类型
      intentFilter.addAction("CustomActionName");
      // 3. 动态注册：调用Context的registerReceiver（）方法
       registerReceiver(mBroadcastReceiver, intentFilter);
   }
  
  // 注册广播后，要在相应位置记得销毁广播
  // 即在onPause() 中unregisterReceiver(mBroadcastReceiver)
  // 当此Activity销毁前，onPause方法一定会执行，所以在此方法中取消注册。
   @Override
   protected void onPause() {
       super.onPause();
        //销毁在onResume()方法中的广播
       unregisterReceiver(mBroadcastReceiver);
       }
  }
  ```

  <span style="color:red">两者区别：</span>

  **静态注册**：常驻型，不受组件生命周期影响。静态注册是在应用安装的时候由系统 PMS（PackageManagerService）完成整个注册过程。

  **动态注册**：非常驻型，广播接收器跟随组件的生命周期变化，有注册就必然得有注销，否则会导致**内存泄露**。对 Activity 最好在 `onResume()` 注册，在 `onPause()` 注销。如果应用退出后，没有撤销已经注册的接收者应用应用将会报错。动态注册由 ContextImpl 来实现的，会调到 AMS.registerReceiver() 注册。

  当广播接收者通过 intent 启动一个 activity 或者 service 时，如果 intent 中无法匹配到相应的组件，动态注册的广播接收者将会导致应用报错，而静态注册的广播接收者将不会有任何报错，因为自从应用安装完成后，广播接收者跟应用已经脱离了关系。

##### 发送广播

**发送普通广播**：定义广播所具备的“意图 (`Intent`) ”，使用 **Context.sendBroadcast()** 方法。

```java
Intent intent = new Intent();
//对应BroadcastReceiver中intentFilter的action
intent.setAction(BROADCAST_ACTION);
//适配7.0及以上静态注册的广播收不到，静态注册需发送显式广播，即给intent指定包名。若是动态注册的则不需要
//intent.setComponent(new ComponentName(getPackageName(),getPackageName()+".CustomBroadcastReceiver"));
//发送广播
sendBroadcast(intent);
```

**发送有序广播**：**Context.sendOrderedBroadcast()**，该方法接收两个参数，Intent & 权限相关的字符串，无特殊要求可置为null。

**发送本地广播**：使用 **LocalBroadcastManager** 类的 sendBroadcast()、registerReceiver()、unregisterReceiver()等方法，用于应用内部传递消息。对于LocalBroadcastManager方式发送的应用内广播，只能通过LocalBroadcastManager动态注册，不能静态注册。

```java
//注册应用内广播接收器
//步骤1：实例化BroadcastReceiver子类 & IntentFilter mBroadcastReceiver 
mBroadcastReceiver = new CustomBroadcastReceiver(); 
IntentFilter intentFilter = new IntentFilter(); 
//步骤2：实例化LocalBroadcastManager的实例
localBroadcastManager = LocalBroadcastManager.getInstance(this);
//步骤3：设置接收广播的类型 
intentFilter.addAction(BROADCAST_ACTION);
//步骤4：调用LocalBroadcastManager单一实例的registerReceiver（）方法进行动态注册 
localBroadcastManager.registerReceiver(mBroadcastReceiver, intentFilter);
//取消注册应用内广播接收器
localBroadcastManager.unregisterReceiver(mBroadcastReceiver);

//发送应用内广播
Intent intent = new Intent();
intent.setAction(BROADCAST_ACTION);
//适配7.0及以上静态注册的广播收不到，静态注册需发送显式广播，即给intent指定包名。若是动态注册的则不需要
//intent.setComponent(new ComponentName(getPackageName(),getPackageName()+".CustomBroadcastReceiver"));
localBroadcastManager.sendBroadcast(intent);
```

#### 四、广播发送和接收原理

广播队列传送广播给 Receiver 的原理其实就是将 BroadcastReceiver 和消息都放到 BroadcastRecord 里面,然后通过 Handler 机制遍历 BroadcastQueue 里面的 BroadcastRecord，将消息发送给 BroadcastReceiver。详细细节可参考：[广播的底层实现原理](https://www.jianshu.com/p/02085150339c)

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghb614kgilj30ng07edg2.jpg)

整个广播的机制可以总结成下面这张图:

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghb5zyxep8j30g00aqaa4.jpg)

### 总结

**使用注意**：

 当系统或应用发出广播时，将会扫描系统中的所有广播接收者，通过 action 匹配将广播发送给相应的接收者，接收者收到广播后将会产生一个广播接收者的实例，运行在 `UI` 线程，执行其中的 onReceiver() 方法，这个实例的生命周期只有10秒，10秒内没有执行结束 onReceiver() 系统将会导致 ANR ；
    在 onReceiver() 执行完毕之后，该实例将会被销毁，所以不要在 onReceiver() 中执行耗时操作，也不要在里面创建子线程处理业务（可能子线程没处理完，接收者就被回收了，子线程也会跟着被回收掉）；正确的处理方法就是通过 intent 调用 Activity 或者 Service 处理业务。

引用文章：

> [Android开发之BroadcastReceiver](https://blog.csdn.net/xubinjie517/article/details/90239812#Sticky_Broadcast_38)
>
> [BroadcastReceiver史上最全面解析](https://www.jianshu.com/p/ca3d87a4cdf3)
>
> [广播的底层实现原理](https://www.jianshu.com/p/02085150339c)

