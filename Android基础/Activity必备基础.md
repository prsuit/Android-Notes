---
title: Activity必备基础
date: 2020-07-22 00:11:45
tags: [Android基础]
categories: [Android基础]
---

### 前言

本文是介绍Android的四大组件之一的Activity。

### 目录

#### 一、Activity是什么

我们都知道android中有四大组件（Activity 活动，Service 服务，BroadcastReceiver 广播接收器，Content Provider 内容提供者），Activity是用的最多也是最基本的组件，Activity 提供窗口来和用户进行交互的组件。官方是这么介绍的：

<!--more-->

> An activity is a single, focused thing that the user can do. Almost all activities interact with the user, so the Activity class takes care of creating a window for you in which you can place your UI with setContentView(View).
>
> 一个activity是一个单独的，用来处理用户操作的窗口。几乎所有的activity都是用来和用户交互的，所以activity类会创建了一个窗口，你可以通过setContentView(View)显示你的UI在窗口上。

- Activity是一个应用程序组件，提供一个屏幕，用户可以用来交互为了完成某项任务。
- Activity中所有操作都与用户密切相关，是一个负责与用户交互的组件，可以通过setContentView(View)来显示指定控件。

#### 二、Activity状态及转换

在android 中，Activity 拥有四种基本状态：

1.**Active/Running**

​	一个新 Activity 启动入栈后，它显示在屏幕最前端，处理是处于栈的最顶端（Activity栈顶），此时它处于可见并可和用户交互的激活状态，叫做活动状态或者运行状态（active or running）。

2.**Paused**

​	当 Activity 失去焦点，被另一个透明或者 Dialog 样式非全屏的 Activity 覆盖时的状态，叫做暂停状态（Paused）。此时它依然与窗口管理器保持连接，系统继续维护其内部状态，所以它仍然可见，但它已经失去了焦点故不可与用户交互。

3.**Stopped**

​	当 Activity 被另外一个 Activity 完全覆盖掉，不可再见时的状态叫做停止状态（Stopped）。

4.**Killed**

​	Activity 被系统杀死回收或者没有被启动时的状态，叫做被杀死的状态（Killed）。

下图说明了 Activity 在不同状态间转换的时机和条件：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gh5lxx3ssej308n081dfu.jpg)

#### 三、Activity栈

Android 是通过一种 Activity 栈的方式来管理 Activity 的，一个 Activity 的实例的状态决定它在栈中的位置。处于前台的 Activity 总是在栈的顶端，当前台的 Activity 因为异常或其它原因被销毁时，处于栈第二层的 Activity 将被激活，上浮到栈顶。当新的 Activity 启动入栈时，原 Activity 会被压入到栈的第二层。一个 Activity 在栈中的位置变化反映了它在不同状态间的转换。Activity 的状态与它在栈中的位置关系如下图所示：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gh5m9bc21zj308w05r748.jpg)

如上所示，除了最顶层即处在 Active 状态的 Activity 外，其它的 Activity 都有可能在系统内存不足时被回收，一个 Activity 的实例越是处在栈的底层，它被系统回收的可能性越大。系统负责管理栈中 Activity 的实例，它根据 Activity 所处的状态来改变其在栈中的位置。

#### 四、Activity生命周期

生命周期函数，常用的7个如下：

**onCreate()**：表示 Activity **正在被创建**，常用来**初始化工作**，比如调用 setContentView 加载界面布局资源，初始化 Activity 所需数据等；

**onReStart()**：表示 Activity **正在重新启动**，当前 Acitivty 从不可见重新变为可见时，onRestart 就会被调用；

**onStart()**：表示 Activity **正在被启动**，此时 Activity **可见但不在前台**，还处于后台，无法与用户交互；

**onResume()**：表示 Activity **获得焦点**，此时 Activity **可见且在前台**并开始活动，位于活动堆栈的顶部，这是与onStart的区别所在；

**onPause()**：表示 Activity **正在停止**，**可见但不在前台**，此时可做一些**存储数据、停止动画**等工作，但是不能太耗时，因为这会影响到新 Activity 的显示，onPause 必须先执行完，新 Activity 的 onResume 才会执行；

**onStop()**：表示 Activity **即将停止**，被新 activity 覆盖了，对用户**不可见**，可以做一些稍微重量级的回收工作，比如注销广播接收器、关闭网络连接等，同样不能太耗时；

**onDestroy()**：表示 Activity **即将被销毁**，这是 Activity 生命周期中的最后一个回调，常做**回收工作、资源释放**；

- 延伸：从**整个生命周期**来看，**onCreate** 和 **onDestroy** 是配对的，分别标识着 Activity 的创建和销毁，并且只可能有**一次调用**； 从 Activity **是否可见**来说，**onStart** 和 **onStop** 是配对的，这两个方法可能被**调用多次**； 从 Activity **是否在前台**来说，**onResume** 和 **onPause** 是配对的，这两个方法可能被**调用多次**； 除了这种区别，在实际使用中没有其他明显区别。

Activity 生命周期循环图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gh5o2g5wxnj30e90if75q.jpg)

**onSaveInstanceState()**：发生条件在系统配置发生改变（例如屏幕方向）导致 Activity 被杀死并重新创建、资源内存不足导致低优先级的 Activity 被杀死，系统会调用 **onSaveInstanceState** 来保存当前 Activity 的状态，此方法调用在 onStop 之前，与 onPause 没有既定的时序关系；

**onRestoreInstanceState()**： 当 Activity 被重建后，系统会调用 **onRestoreInstanceState**，并且把onSaveInstanceState 方法所保存的 Bundle 对象**同时传参**给 onRestoreInstanceState() 和 onCreate() ，因此可以通过这两个方法判断 Activity **是否被重建**，调用在 onStart 之后，可使用 onSaveInstanceState() 和onRestoreInstanceState()（或onCreate()）来保存和恢复 Activity 活动状态。

#### 五、Activity启动和通信

**Intent**是一种消息传递的机制，它负责对操作的动作、动作涉及数据、附加数据进行描述。Android 则根据此Intent的描述，负责找到对应的组件，将 Intent 传递给调用的组件，并完成组件的调用。

##### 显式Intent

通过指定具体的组件类，通知应用启动对应的组件。

```java
Intent intent = new Intent();
intent.setClass(MainActivity.this,SecondAcvivity.class);
intent.putExtra("values","传个值");
startActivity(intent);
```

##### 隐式Intent

通过指定了一系列更为抽象的 action 和 category 等信息，然后交由系统去分析这个 Intent，并帮我们找出合适的活动去启动。

```java
Intent intent = new Intent("com.example.administrator.ACTION_START");
startActivity(intent);
```

要在清单文件中`<activity>`标签下配置`<intent-filter>`的内容，可以指定当前活动能够响应的 action
和 category，如下：

```java
<activity android:name=".SecondActivity" >
	<intent-filter>
		<action android:name="com.example.administrator.ACTION_START" />
		<category android:name="android.intent.category.DEFAULT" />
	</intent-filter>
</activity>
```

##### Intent Filter

描述了一个组件愿意接收什么样的 Intent 对象，Android 将其抽象为 android.content.IntentFilter 类。 Activity 通过指定其 Intent Filter告诉系统该 Activity 可以响应什么类型的 Intent，有三大属性Action、URL、Category可以配置。

Activity 中 Intent Filter 匹配的过程：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gh6iwqed6kj30890bvmxc.jpg)

**Action 匹配**

是一个用户定义的字符串，一个 Intent Filter 可以包含多个 Action，用于标示 Activity 所能接受的”动作”。

```java
<intent-filter >
 	<action android:name="android.intent.action.MAIN" />
 	<action android:name="com.xx.myaction" />
	......
 </intent-filter>
```

**URI 数据匹配**

通过 URI 携带外部数据给目标组件，在`<intent-filter>`标签中配置一个`<data>`标签。

mimeType 属性指定携带外部数据的数据类型，scheme 指定协议，host、port、path 指定数据的位置、端口、和路径。

```java
<intent-filter>
	<data android:mimeType="mimeType" android:scheme="scheme"
 				android:host="host" android:port="port" android:path="path"/>
</intent-filter
```

只有`<data>`标签中指定的内容和 Intent 中携带的 Data 完全一致时，当前活动才能够响应该 Intent。不过一般在`<data>`标签中都不会指定过多的内容。

**Category 类别匹配**

为组件定义一个 Category 类别列表，当 Intent 中包含这个列表的所有项目时 Category 类别匹配才会成功。

```java
<intent-filter>
	<category android:name="android.intent.category.DEFAULT" />
</intent-filter>
```

android.intent.category.DEFAULT 是一种默认的 category，在调用startActivity()方法的时候会自动将这个 category 添加到 Intent 中。

**接收上一个Activity返回数据**

**startActivityForResult()**方法也是用于启动 Activity 的，这个方法可以 Activity 销毁的时候能够返回一个结果给上一个活动。该方法接收两个参数，第一个参数还是 Intent，第二个参数是请求码，用于在之后的回调中判断数据的来源。

```java
Intent intent = new Intent(MainActivity.this, SecondActivity.class);
startActivityForResult(intent, 1);
```

返回代码：

```java
Intent intent = new Intent();
intent.putExtra("data_return", "Hello MainActivity");
setResult(RESULT_OK, intent);
finish();
```

**setResult()**方法接收两个参数，第一个参数用于向上一个活动返回处理结果，一般只使用**RESULT_OK**或**RESULT_CANCELED**，第二个参数则是把带有数据的 Intent 传递回去，然后调用了 finish()方法来销毁当前活动。

在需要接收返回数据的 Activity 中重写**onActivityResult()**方法，

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
switch (requestCode) {
	case 1:
	if (resultCode == RESULT_OK) {
		String returnedData = data.getStringExtra("data_return");
		Log.d("FirstActivity", returnedData);
	}
	break;
	default:
	}
}
```

onActivityResult()方法带有三个参数：

第一个参数 requestCode，即我们在启动活动时传入的请求码。

第二个参数 resultCode，即我们在返回数据时传入的处理结果。

第三个参数 data，即携带着返回数据的 Intent。

#### 六、Activity启动模式

##### 使用 manifest 文件

在 manifest 文件中通过给`<activity>`标签的**android:launchMode**属性来指定恰当的启动模式。

1.**standard标准模式**

​		每次启动一个 Activity 都会重新创建一个新的实例，不管这个实例是否已经存在，此模式的 Activity 默认会进入启动它的 Activity 所属的任务栈中；

2.**singleTop栈顶复用模式**

​		如果新 Activity 已经位于任务栈的栈顶，那么此 Activity 不会被重新创建，同时会回调 **onNewIntent** 方法，如果已经存在但不在栈顶，那么 Activity 依然会被重新创建；一般用于推送消息跳转界面。

3.**singleTask栈内复用模式**

​		Activity 在同一个Task内只有一个实例。每次启动该 Activity 时系统首先会在返回栈中检查是否存在该Activity 的实例，如果发现已经存在则直接使用该实例，回调其**onNewIntent**方法，并把在这个 Activity 之上的所有 Activity 统统出栈；如果没有发现就会创建一个新的 Activity 实例，并将其加入Task栈顶，一般项目的主页面用该启动模式。

4.**singleInstance单实例模式**

Activity只能单独地位于一个任务栈中，且此任务栈中只有唯一一个实例，在不同app之间的共享活动实例，一般用于系统功能界面。

##### 使用 Intent 标志

Activity常用的标记位Flags

- **FLAG_ACTIVITY_NEW_TASK**：对应 singleTask 启动模式。
- **FLAG_ACTIVITY_SINGLE_TOP**：对应 singleTop 启动模式。
- **FLAG_ACTIVITY_CLEAR_TOP**：在同一个任务栈中所有位于它上面的 Activity 都要出栈。这个标记位一般会和 singleTask 模式一起出现，在这种情况下，被启动Activity的实例如果已经存在，那么系统就会回调onNewIntent。如果被启动的 Activity 采用 standard 模式启动，那么它以及连同它之上的 Activity 都要出栈，系统会创建新的 Activity 实例并放入栈中。
- **FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS :** 具有这个标记的 Activity 不会出现在历史 Activity 列表中。

### 总结

以此文记录 Android 中 Activity 的基础知识，便于对 Activity 的理解和学习。

