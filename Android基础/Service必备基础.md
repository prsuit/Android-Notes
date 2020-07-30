---
title: Service必备基础
date: 2020-07-25 19:21:49
categories: [Android基础]
tags: [Android基础]
---

### 前言

本文是介绍Android的四大组件之一的Service。

### 目录

#### 一、什么是Service

Service 是一个应用程序组件，它能在后台执行一些耗时较长的操作，并且不提供用户界面。服务能被其他应用程序的组件启动，即使用户切换到另外的应用时还能保持后台运行。此外，应用程序组件还能与服务绑定，并与服务进行交互，甚至能进行进程间通信（IPC）。 比如，服务可以处理网络传输、音乐播放、执行文件I/O、或者与 content provider 进行交互，所有这些都是后台进行的。

<!--more-->

- 运行完全不依赖UI的，只要进程还在，Service 就可以继续运行，可以和其他组件组件交互；
- 一般运行在与创建服务时所在的应用程序进程中，要运行在单独的进程当中，在声明时需指定 `android:process`；

- 服务不会自动开启线程，我们需要在服务的内部手动创建子线程，并在这里执行具体的耗时任务。

#### 二、Service分类

##### 按运行分类

**前台服务**：指那些经常会被用户关注的服务，因此内存过低时它不会成为被杀的对象。 前台服务必须提供一个状态栏通知，比较不容易被系统回收。

**后台服务**：指在后台默默工作，提供数据运算等的服务，它优先级还是比较低的，当系统出现内存不足情况时，就有可能会回收掉正在后台运行的Service。

##### 按使用分类

**本地服务**：用于应用程序内部，实现一些耗时任务，并不占用应用程序所属线程，而是单开线程后台执行。 

**远程服务**：用于 Android 系统内部的应用程序之间，可被其他应用程序复用，比如天气预报服务，其他应用程序不需要再写这样的服务，调用已有的即可。

#### 三、Service启动状态

##### 启动状态(Started)

当应用组件（如 Activity）通过调用 **startService()** 启动服务时，服务即处于“启动”状态。一旦启动，服务即可在后台无限期运行，即使启动服务的组件已被销毁也不受影响，除非外部手动调用 **stopService()** 或内部调用 **stopSelf()** 才能停止服务， 已启动的服务通常是执行单一操作，而且不会将结果返回给调用方。

##### 绑定状态(Bound)

当应用组件通过调用 **bindService()** 绑定到服务时，服务即处于“绑定”状态。绑定服务提供了一个客户端-服务器 **IBinder** 接口，允许组件与服务进行交互、发送请求、获取结果，甚至是利用进程间通信 (IPC) 跨进程执行这些操作。 仅当与另一个应用组件绑定时，绑定服务才会运行。 多个组件可以同时绑定到该服务，但全部取消绑定后，该服务即会被销毁。

#### 四、Service生命周期

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gh7ysawqzfj30at0e3abt.jpg)

**onCreate()**：首次创建服务时，将调用此方法，如果服务已在运行，则不会调用此方法，该方法只调用一次；

**onStartComand()**：服务通过 **startService()** 启动时会调用，多次执行 **startService()** 方法，该方法也会相应的多次调用；

**onBind()**：服务通过 **bindService()** 启动且服务是第一次创建时会调用，在此方法必须返回 一个 IBinder 接口的实现类对象，供客户端用来与服务进行通信，服务在启动状态的情况下可返回 null ；

**onUnBind()**：服务通过 **unbindService()** 被解绑时调用；

**onDestroy()**：服务停止或被解绑后调用；

#### 五、Service使用

##### Service在清单文件中的声明

通过继承Service基类自定义而来，都需要在AndroidManifest.xml中声明，Service在AndroidManifest.xml中的声明语法，其格式如下(不是所有都必填)：

```java
<service android:enabled=["true" | "false"]
    android:exported=["true" | "false"]
    android:icon="drawable resource"
    android:isolatedProcess=["true" | "false"]
    android:label="string resource"
    android:name="string"
    android:permission="string"
    android:process="string" >
    . . .
</service>
```

- android:exported：代表是否能被其他应用隐式调用，其默认值是由 service 中有无 intent-filter 决定的，如果有 intent-filter ，默认值为 true ，否则为 false 。为 false 的情况下，即使有 intent-filter 匹配，也无法打开，即无法被其他应用隐式调用。
- android:name：对应 Service 类名，唯一必需的属性。
- android:permission：申明此服务的权限。
- android:process：是否需要在单独的进程中运行，当设置为 android:process=”:remote” 时，代表Service 在单独的进程中运行。注意“：”很重要，它的意思是指要在当前进程名称前面附加上当前的包名，所以 “remote” 和 ”:remote” 不是同一个意思，前者的进程名称为：remote，而后者的进程名称为：App-packageName:remote。
- android:isolatedProcess ：设置 true 意味着，服务会在一个特殊的进程下运行，这个进程与系统其他进程分开且没有自己的权限。与其通信的唯一途径是通过服务的API(bind and start)。
- android:enabled：是否可以被系统实例化，默认为 true因为父标签 也有 enable 属性，所以必须两个都为默认值 true 的情况下服务才会被激活，否则不会激活。

##### 通过startService启动

- service 会一直无限期运行下去，只有外部调用了 stopService() 或 stopSelf() 方法时，该 Service 才会停止运行并销毁。

- 多次 startService 不会重复执行onCreate回调，但每次都会执行 onStartCommand 回调

- **onStartCommand()** 方法的返回值int类型才值得注意的，它有三种可选值：

  START_STICKY：“黏性的”，当 Service 因内存不足而被系统 kill 后，一段时间后内存再次空闲时，系统将会尝试重新创建此 Service ，一旦创建成功后将回调 onStartCommand 方法，但其中的 Intent 将是null，除非有挂起的Intent，如pendingintent，比较适用于不执行命令、但无限期运行并等待作业的媒体播放器或类似服务。

  START_NOT_STICKY：“非黏性的”，当 Service 因内存不足而被系统kill后，即使系统内存再次空闲时，系统也不会尝试重新创建此Service。

  START_REDELIVER_INTENT：当 Service 因内存不足而被系统kill后，则会重建服务，并通过传递给服务的最后一个 Intent 调用 onStartCommand()，任何挂起 Intent 均依次传递。与 START_STICKY 不同的是，其中的传递的 Intent 将是非空，是最后一次调用 startService 中的 intent 。这个值适用于主动执行应该立即恢复的作业（例如下载文件）的服务。

##### 通过bindService启动

- 该启动方式的调用者和服务之间是典型的 **client-server** 模式。调用者是 client ，service 则是 server 端。**service 只有一个**，但绑定到 service 上面的 client 可以有一个或很多个。这里所提到的 client 指的是组件，比如某个Activity。

- **client 可以通过 IBinder 接口获取 Service 实例**，从而实现在 client 端直接调用 Service 中的方法以实现交互。

- 启动服务的生命周期与其绑定的 client 息息相关。当 client 销毁时，client 会自动与 Service 解除绑定。当然，client 也可以明确调用 Context的 **unbindService()** 方法与 Service 解除绑定。**当没有任何 client 与 Service 绑定时，Service 会自行销毁**。

- **onBind()** 方法必须返回一个 IBinder接口的实现类对象，该类用以提供客户端用来与服务进行交互的编程接口，该接口可以通过三种方法定义接口：

  **扩展 Binder 类** ：如果服务是提供给自有应用专用的，并且 Service (服务端)与客户端相同的进程中运行，即客户端和服务位于**同一应用和进程**内才有效。通过扩展 Binder 类并从 onBind() 返回它的一个实例来创建接口。客户端收到 Binder 后，可利用它直接访问 Binder 实现中以及 Service 中可用的公共方法。如果我们的服务只是自有应用的后台工作线程，则优先采用这种方法。不采用该方式创建接口的唯一原因是，服务被其他应用或不同的进程调用。

  **使用 Messenger** ：通过它可以在**不同的进程**中共传递Message对象(Handler中的Messager，因此 Handler 是 Messenger 的基础)，在Message中可以存放我们需要传递的数据，然后在进程间传递。因为 Messenger 会在单一线程中创建包含所有请求的队列，也就是说Messenger是以串行的方式处理客户端发来的消息，这样我们就不必对服务进行线程安全设计了。

  **使用 AIDL** ：由于 Messenger 是以串行的方式处理客户端发来的消息，如果当前有大量消息同时发送到Service(服务端)，Service 仍然只能一个个处理，这也就是 Messenger 跨进程通信的缺点了，因此如果有大量并发请求，Messenger 就会显得力不从心了，这时 AIDL（Android Interface Definition Language） Android 接口定义语言就派上用场了，但实际上 Messenger 的跨进程方式其底层实现就是AIDL，只不过 android 系统帮我们封装成透明的 Messenger 罢了。如果我们想让服务同时处理多个请求，则应该使用 AIDL。它可以用于让某个 Service 与多个应用程序组件之间进行跨进程通信，从而可以实现**多个应用程序共享同一个 Service** 的功能。

```java
//服务端LocalService部分代码
public class LocalService extends Service{
	private MyBinder mBinder = new MyBinder();
		@Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
  ...
  
  //建Binder对象，返回给客户端使用，提供数据交换的接口
	class MyBinder extends Binder {
 	    // 声明一个方法，getService。（提供给客户端调用）
    LocalService getService() {
    // 返回当前对象LocalService,这样我们就可在客户端调用Service的公共方法了
      return LocalService.this;
    }
      
    //定义交互方法
    public void getCount() {
      
    }
 	}
}
```

```java
//客户端--绑定服务实例
public class BindActivity extends Activity {
  private ServiceConnection conn;
  private LocalService.MyBinder mBinder;
  
  @Overrid
  protected void onCreate(Bundle savedInstanceState){
     super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_bind);
        btnBind = (Button) findViewById(R.id.BindService);
        btnUnBind = (Button) findViewById(R.id.unBindService);
        btnGetDatas = (Button) findViewById(R.id.getServiceDatas);
        //创建绑定对象
        final Intent intent = new Intent(this, LocalService.class);
        // 开启绑定
        btnBind.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d(TAG, "绑定调用：bindService");
                //调用绑定方法
                bindService(intent, conn, Service.BIND_AUTO_CREATE);
            }
        });
        // 解除绑定
        btnUnBind.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.d(TAG, "解除绑定调用：unbindService");
                // 解除绑定
                if(mBinder!=null) {
                    mBinder = null;
                    unbindService(conn);
                }
            }
        });
      // 获取数据
        btnGetDatas.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                if (mBinder != null) {
                    // 通过绑定服务传递的Binder对象，获取Binder暴露出来的数据
                    Log.d(TAG, "从服务端获取数据：" + mBinder.getCount());
                } else {
                    Log.d(TAG, "还没绑定呢，先绑定,无法从服务端获取数据");
                }
            }
        });
       conn = new ServiceConnection() {
            /**
             * 与服务器端交互的接口方法 绑定服务的时候被回调，在这个方法获取绑定Service传递过来的IBinder对象，
             * 通过这个IBinder对象，实现宿主和Service的交互。
             */
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                Log.d(TAG, "绑定成功调用：onServiceConnected");
                // 获取Binder
                mBinder = (LocalService.LocalBinder) service;
            }
            /**
             * 当取消绑定的时候被回调。但正常情况下是不被调用的，它的调用时机是当Service服务被意外销毁时，
             * 例如内存的资源不足时这个方法才被自动调用。
             */
            @Override
            public void onServiceDisconnected(ComponentName name) {
                mBinder = null;
            }
        };

  }
}
```

- **onServiceConnected(ComponentName name, IBinder service)** 
  服务绑定成功后系统会调用该方法以传递服务的 onBind() 方法返回的 IBinder。其中 service 便是服务端返回的 IBinder 实现类对象，ComponentName 是一个封装了组件信息的类。

- **onServiceDisconnected(ComponentName name)** 
  系统会在与服务的连接意外中断时（例如**当服务崩溃或被终止**时）调用该方法。<span style="color:red">注意:当客户端取消绑定时，系统“绝对不会”调用该方法</span>。

- **bindService(Intent service, ServiceConnection conn, int flags)**

  其中 Intent 是我们要绑定的服务的意图，而 ServiceConnection 代表与服务的连接，它只有两个方法，前面已分析过，flags 则是指定绑定时是否自动创建 Service 。0代表不自动创建、BIND_AUTO_CREATE 则代表自动创建。

- **unbindService(ServiceConnection conn)** 
  该方法执行解除绑定的操作，其中 ServiceConnection 代表与服务的连接。

#### 六、启动服务与绑定服务间的转换问题

​		虽然服务的状态有启动和绑定两种，但实际上一个服务可以同时是这两种状态，它既可以是启动服务（以无限期运行），也可以是绑定服务。有点需要注意的是 Android 系统仅会为一个 Service 创建一个实例对象，所以不管是启动服务还是绑定服务，操作的是同一个 Service 实例，而且由于绑定服务或者启动服务执行顺序问题将会出现以下两种情况：**（启动服务的优先级比绑定服务高一些）**

- **先绑定服务后启动服务**

  如果当前 Service 实例先以绑定状态运行，然后再以启动状态运行，那么绑定服务将会转为启动服务运行，这时如果之前绑定的宿主（Activity）被销毁了，也不会影响服务的运行，服务还是会一直运行下去，指定收到调用停止服务或者内存不足时才会销毁该服务。

- **先启动服务后绑定服务**

  如果当前 Service 实例先以启动状态运行，然后再以绑定状态运行，当前启动服务并不会转为绑定服务，但是还是会与宿主绑定，只是即使宿主解除绑定后，服务依然按启动服务的生命周期在后台运行，直到有 Context调用了 stopService() 或是服务本身调用了 stopSelf() 方法抑或内存不足时才会销毁服务。

#### 七、前台服务

前台服务被认为是用户主动意识到的一种服务，因此在内存不足时，系统也不会考虑将其终止。前台服务必须为状态栏提供通知，设置服务运行于前台的方法。

##### 设置服务运行于前台

- **startForeground(int id, Notification notification)**

  在 Service 创建 onCreate 时调用，该方法的作用是把当前服务设置为前台服务，其中id参数代表唯一标识通知的整型数，需要注意的是提供给 startForeground() 的整型 ID 不得为 0，而notification是一个状态栏的通知。Android 8.0之后对于通知栏也进行了整改，增加了通知渠道，通知组的概念。

- **stopForeground(boolean removeNotification)** 

  在 Service 销毁时调用，该方法是用来从前台删除服务，removeNotification 是否也删除状态栏通知。

##### 启动前台服务

在Android 8.0 之后需要使用 **startForegroundService()** 启动前台服务，并在清单文件中添加前

台服务权限 `android.permission.FOREGROUND_SERVICE` 。

```java
 if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        startForegroundService(mForegroundService);
    } else {
        startService(mForegroundService);
    }
```

#### 八、如何提高服务在后台运行时间

关于 Service 保活后期会整理一篇主流的具体解决方案，主要思路是：

1.**用 startService 方式启动 Service ，onStartCommand 方式中，返回 START_STICKY** 。

2.**提高 Service 的优先级**，在清单文件中对于 intent-filter 可以通过 android:priority = "1000" 属性设置，1000是最高值，如果数字越小则优先级越低。

3.**提升 Service 进程的优先级**，使用前台服务。

4.**设置应用白名单和电量管理**。

### 总结

- Service 实例对象同一应用中同时只会有一个。
- Service默认并不会运行在子线程中，也不运行在一个独立的进程中，它同样执行在主线程中（UI线程）。换句话说，不要在Service里执行耗时操作，除非手动打开一个子线程，否则有可能出现主线程被阻塞（ANR）的情况。
- 使用了远程 Service 后，Service 已经在另外一个进程当中运行了，所以只会阻塞该进程中的主线程，并不会影响到当前的应用程序。
- 默认 Service 和创建者在同一个进程内，如果将 Service 的 android:process 属性指定成" :remote"，表示Service 是在单独的进程创建远程服务。调用 startService 可以启动服务，但不能调用 bindService 绑定服务，因为创建者和 Service 运行在两个不同的进程中，不能再使用传统的建立关联的方式，需要使用 AIDL 来跨进程通信才可以绑定。
- 只有Activity、Service、Content Provider 能够绑定服务；BroadcastReceiver 广播接收器不能绑定服务

引用文章：

> [安卓Service 详解](https://blog.csdn.net/hdhhd/article/details/80612726)
>
> [Android Service两种启动方式详解](https://www.jianshu.com/p/4c798c91a613)
>
> [Android Service完全解析，关于服务你所需知道的一切](http://blog.csdn.net/guolin_blog/article/details/11952435)
>
> [Android 基于Message的进程间通信 Messenger完全解析](https://blog.csdn.net/lmj623565791/article/details/47017485)
>
> [Android aidl Binder框架浅析](https://blog.csdn.net/lmj623565791/article/details/38461079)

