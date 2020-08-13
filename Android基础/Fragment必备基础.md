---
title: Fragment必备基础
categories:
  - Android基础
tags:
  - Android基础
date: 2020-07-31 20:58:51
---

### 前言

本文是介绍Android常用的组件之一的Fragment。

### 目录

#### 一、什么是Fragment

Fragment，简称碎片，是 Android最基本，最重要的基础概念之一。Fragment 官方的定义是：

> A Fragment represents a behavior or a portion of user interface in an Activity. You can combine multiple fragments in a single activity to build a multi-pane UI and reuse a fragment in multiple activities. You can think of a fragment as a modular section of an activity, which has its own lifecycle, receives its own input events, and which you can add or remove while the activity is running.

<!--more-->

- Fragment 是依赖于 Activity 的，不能独立存在的。
- 一个 Activity 里可以有多个 Fragment 。
- 一个 Fragment 可以被多个 Activity 重用。
- Fragment 有自己的生命周期，并能接收输入事件。
- 我们能在 Activity 运行时动态地添加或删除 Fragment 。

**Fragment 的优势有以下几点：**

- **模块化(Modularity)**：我们不必把所有代码全部写在Activity中，而是把代码写在各自的Fragment中。
- **可重用(Reusability)**：多个Activity可以重用一个Fragment。
- **可适配(Adaptability)**：根据硬件的屏幕尺寸、屏幕方向，能够方便地实现不同的布局，这样用户体验更好。

**Fragment核心的类有：**

- **Fragment**：Fragment 的基类，任何创建的 Fragment 都需要继承该类。
- **FragmentManager**：管理和维护 Fragment。它是抽象类，具体的实现类是 **FragmentManagerImpl**。
- **FragmentTransaction**：对 Fragment 的添加、删除等操作都需要通过事务方式进行。它是抽象类，具体的实现类是 **BackStackRecord**。

#### 二、生命周期

Fragment的生命周期和Activity类似，但比Activity的生命周期复杂一些，基本的生命周期方法如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghguz2jnu5j30b409uq34.jpg)

- **onAttach**：Fragment 和 Activity 相关联时调用。可以通过该方法获取 Activity 引用，还可以通过getArguments() 获取参数。
- **onCreate**：Fragment 被创建时调用。
- **onCreateView**：创建 Fragment 的布局。
- **onActivityCreated**：当 Activity 完成 onCreate() 时调用，在 onCreateView() 方法之后调用该方法。
- **onStart**：当 Fragment 开始可见时调用，此时还不可交互。
- **onResume**：当 Fragment 可见且可交互时调用。
- **onPause**：当 Fragment 不可交互但可见时调用，表明用户将要离开当前 Fragment 。
- **onStop**：当 Fragment 不可见时调用，Fragment 被停止。
- **onDestroyView**：当 Fragment 的 UI 从视图结构中移除时调用，销毁与 Fragment 有关的视图。
- **onDestroy**：销毁 Fragment 时调用。
- **onDetach**：当 Fragment 和 Activity 解除关联时调用。

当一个 fragment 被创建的时候，需调用以下生命周期方法：onAttach(), onCreate(), onCreateView(), onActivityCreated()；

当这个 fragment 对用户可见可交互的时候，需调用：onStart() ,onResume()；

当这个 fragment 对用户不可交互不可见，进入后台模式需调用：onPause(),onStop()；

当这个 fragment 被销毁或者是持有它的 Activity 被销毁了，调用：onPause() ,onStop(), onDestroyView(), onDestroy(),onDetach()。

因为 Fragment 是依赖 Activity 的，Fragment 和 Activity 的生命周期方法有密切的关系和顺序，如图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghgwq0ptw2j30hs0l4jsp.jpg)

举例来理解Fragment生命周期方法，共有两个Fragment：F1和F2，F1在初始化时就加入Activity，点击F1中的按钮调用replace替换为F2。

当F1在Activity的`onCreate()`中被添加时，日志如下：

```java
BasicActivity: [onCreate] BEGIN
BasicActivity: [onCreate] END
BasicActivity: [onStart] BEGIN
Fragment1: [onAttach] BEGIN 
Fragment1: [onAttach] END
BasicActivity: [onAttachFragment] BEGIN
BasicActivity: [onAttachFragment] END
Fragment1: [onCreate] BEGIN
Fragment1: [onCreate] END
Fragment1: [onCreateView]
Fragment1: [onViewCreated] BEGIN
Fragment1: [onViewCreated] END
Fragment1: [onActivityCreated] BEGIN
Fragment1: [onActivityCreated] END
Fragment1: [onStart] BEGIN
Fragment1: [onStart] END
BasicActivity: [onStart] END
BasicActivity: [onPostCreate] BEGIN
BasicActivity: [onPostCreate] END
BasicActivity: [onResume] BEGIN
BasicActivity: [onResume] END
BasicActivity: [onPostResume] BEGIN
Fragment1: [onResume] BEGIN
Fragment1: [onResume] END
BasicActivity: [onPostResume] END
BasicActivity: [onAttachedToWindow] BEGIN
BasicActivity: [onAttachedToWindow] END
```

从上图可以看出：

- Fragment 的 onAttach()->onCreate()->onCreateView()->onActivityCreated()->onStart()都是在 Activity 的 onStart() 中调用的。

- Fragment 的 onResume() 在 Activity 的 onResume() 之后调用。

当点击F1的按钮，调用`replace()`替换为F2，且不加`addToBackStack()`时，日志如下：

```java
Fragment2: [onAttach] BEGIN
Fragment2: [onAttach] END
BasicActivity: [onAttachFragment] BEGIN
BasicActivity: [onAttachFragment] END
Fragment2: [onCreate] BEGIN
Fragment2: [onCreate] END
Fragment1: [onPause] BEGIN
Fragment1: [onPause] END
Fragment1: [onStop] BEGIN
Fragment1: [onStop] END
Fragment1: [onDestroyView] BEGIN
Fragment1: [onDestroyView] END
Fragment1: [onDestroy] BEGIN
Fragment1: [onDestroy] END
Fragment1: [onDetach] BEGIN
Fragment1: [onDetach] END
Fragment2: [onCreateView]
Fragment2: [onViewCreated] BEGIN
Fragment2: [onViewCreated] END
Fragment2: [onActivityCreated] BEGIN
Fragment2: [onActivityCreated] END
Fragment2: [onStart] BEGIN
Fragment2: [onStart] END
Fragment2: [onResume] BEGIN
Fragment2: [onResume] END
```

可以看到，F1最后调用了`onDestroy()`和`onDetach()`。

当点击F1的按钮，调用`replace()`替换为F2，且加`addToBackStack()`时，日志如下：

```java
Fragment2: [onAttach] BEGIN
Fragment2: [onAttach] END
BasicActivity: [onAttachFragment] BEGIN
BasicActivity: [onAttachFragment] END
Fragment2: [onCreate] BEGIN
Fragment2: [onCreate] END
Fragment1: [onPause] BEGIN
Fragment1: [onPause] END
Fragment1: [onStop] BEGIN
Fragment1: [onStop] END
Fragment1: [onDestroyView] BEGIN
Fragment1: [onDestroyView] END
Fragment2: [onCreateView]
Fragment2: [onViewCreated] BEGIN
Fragment2: [onViewCreated] END
Fragment2: [onActivityCreated] BEGIN
Fragment2: [onActivityCreated] END
Fragment2: [onStart] BEGIN
Fragment2: [onStart] END
Fragment2: [onResume] BEGIN
Fragment2: [onResume] END
```

可以看到，F1被替换时，最后只调到了`onDestroyView()`，并没有调用`onDestroy()`和`onDetach()`。当用户点返回按钮回退事务时，F1会调 onCreateView()->onActivityCreated()->onStart()->onResume()，因此在 Fragment 事务中加不加`addToBackStack()`会影响 Fragment 的生命周期。

FragmentTransaction 有一些基本方法，下面给出调用这些方法时，Fragment 生命周期的变化：

- add(): onAttach()->…->onResume()，添加进来的 fragment 都是可见的，不能重复添加同一 fragment，切换 fragment 时，不销毁当前的，创建新的，不会重新创建，一般配合 hide 或 remove 使用。
- remove(): onPause()->…->onDetach()，销毁 fragment 。
- replace(): 相当于旧 Fragment 调用remove()，新 Fragment 调用 add()，切换 fragment 时每次都会销毁当前的，创建新的，会重新创建初始化。和 add() 区别：**是否要销毁当前fragment清空容器再添加新fragment**
- show(): 不调用任何生命周期方法，调用该方法的前提是要显示的 Fragment 已经被添加到容器，只是纯粹把 Fragment UI 的 setVisibility 为 true。
- hide(): 不调用任何生命周期方法，调用该方法的前提是要显示的 Fragment 已经被添加到容器，只是纯粹把Fragment UI的setVisibility为false。
- detach(): onPause()->onStop()->onDestroyView()。UI从布局中移除，但是仍然被 FragmentManager 管理。
- attach(): onCreateView()->onStart()->onResume()。

#### 三、基本使用

##### 创建 Fragment

首先创建继承 Fragment 的类，名为Fragment1：

```java
public class Fragment1 extends Fragment{  
  private static String ARG_PARAM = "param_key"; 
     private String mParam; 
     private Activity mActivity; 
     public void onAttach(Context context) {
        mActivity = (Activity) context;
        mParam = getArguments().getString(ARG_PARAM);  //获取参数
    }
     public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View root = inflater.inflate(R.layout.fragment_1, container, false);
        TextView view = root.findViewById(R.id.text);
        view.setText(mParam);
             return root;
    }    
     public static Fragment1 newInstance(String str) {
        Fragment1 frag = new Fragment1();
        Bundle bundle = new Bundle();
        bundle.putString(ARG_PARAM, str);
        fragment.setArguments(bundle);   //设置参数
        return fragment;
    }
}
```

Fragment 有很多可以复写的方法，其中最常用的就是`onCreateView()`，该方法返回 Fragment 的UI布局，需要注意的是`inflate()`的第三个参数是 **false** ，因为在 Fragment 内部实现中，会把该布局添加到 container 中，如果设为 true，那么就会重复做两次添加，则会抛出异常：

```java
Caused by: java.lang.IllegalStateException: The specified child already has a parent. You must call removeView() on the child's parent first.
```

如果在创建Fragment时要传入参数，必须要通过`setArguments(Bundle bundle)`方式添加，而不建议通过为Fragment 添加带参数的构造函数，因为通过`setArguments()`方式添加，在由于内存紧张导致Fragment被系统杀掉并恢复（re-instantiate）时能保留这些数据。官方建议如下：

```java
It is strongly recommended that subclasses do not have other constructors with parameters, since these constructors will not be called when the fragment is re-instantiated.
```

我们可以在 Fragment 的`onAttach()`中通过`getArguments()`获得传进来的参数，并在之后使用这些参数。如果要获取 Activity 对象，不建议调用`getActivity()`，而是在`onAttach()`中将 Context 对象强转为 Activity 对象。

##### 添加Fragment到Activity

- **静态添加**：通过 `<fragment>` 标签的形式添加到 Activity 的布局 xml 当中，缺点是一旦添加就不能在运行时删除。

- **动态添加**：通过 java 代码将 fragment 添加到宿主 Activity 中，这种方式比较灵活，常用这种方式。

这里只给出动态添加的方式。首先Activity需要有一个容器存放 Fragment ，一般是 FrameLayout，因此在Activity 的布局文件中加入 FrameLayout：

```java
<FrameLayout
    android:id="@+id/container"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

然后在`onCreate()`中，通过以下代码将 Fragment 添加进 Activity 中。

```java
if (bundle == null) {
    getSupportFragmentManager().beginTransaction()
        .add(R.id.container, Fragment1.newInstance("hello world"),"f1")
      //.addToBackStack("fname")
        .commit();
}
```

注意几点：

- 使用`getSupportFragmentManager()`获取 FragmentManager。
- `add()`是对 Fragment 众多操作中的一种，还有`remove()`, `replace()`等，第一个参数是根容器的id（FrameLayout的id，即”@id/container”），第二个参数是 Fragment 对象，第三个参数是 fragment 的 tag 名，指定 tag 的好处是后续我们可以通过`Fragment1 frag = getSupportFragmentManager().findFragmentByTag("f1")`从 FragmentManager 中查找 Fragment 对象。
- 在一次事务中，可以做多个操作，比如同时做`add().remove().replace()`。
- `commit()`操作是异步的，内部通过`mManager.enqueueAction()`加入处理队列。对应的同步方法为`commitNow()`，`commit()`内部会有`checkStateLoss()`操作，如果开发人员使用不当（比如`commit()`操作在`onSaveInstanceState()`之后），可能会抛出异常，而`commitAllowingStateLoss()`方法则是不会抛出异常版本的`commit()`方法，但是尽量使用`commit()`，而不要使用`commitAllowingStateLoss()`。
- `addToBackStack("fname")`是可选的。FragmentManager 拥有回退栈（BackStack），类似于 Activity 的任务栈，如果添加了该语句，就把该事务加入回退栈，当用户点击返回按钮，会回退该事务（回退指的是如果事务是`add(frag1)`，那么回退操作就是`remove(frag1)`）；如果没添加该语句，用户点击返回按钮会直接销毁Activity。
- Fragment 有一个常见的问题，即 Fragment 重叠问题，这是由于页面发生销毁重建(旋转屏幕、内存不足等情况被强杀重启)回到前台，重新初始化时，再次将 fragment 加入 activity，而 FragmentActivity 帮我们保存了 Fragment 的状态，并且在页面重启后会帮我们恢复，因此可在Activity的`onCreate()`里加上页面是否是重启的判断`if(saveInstanceState == null){}`。

**Fragment有个常见的异常：**

```java
java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
    at android.support.v4.app.FragmentManagerImpl.checkStateLoss(FragmentManager.java:1341)
    at android.support.v4.app.FragmentManagerImpl.enqueueAction(FragmentManager.java:1352)
    at android.support.v4.app.BackStackRecord.commitInternal(BackStackRecord.java:595)
    at android.support.v4.app.BackStackRecord.commit(BackStackRecord.java:574)
```

该异常出现的原因是：commit() 在 onSaveInstanceState() 后调用。首先，onSaveInstanceState() 是在 Activity 有可能被系统回收的异常终止情况下，而且是在 onPause() 之后，onStop() 之前调用。onRestoreInstanceState() 在onStart() 之后，onResume() 之前。

**因此避免出现该异常的方案有：**

- 不要把 Fragment 事务放在异步线程的回调中，比如不要把 Fragment 事务放在 AsyncTask 的onPostExecute()，因此 onPostExecute() 可能会在 onSaveInstanceState() 之后执行。
- 逼不得已时使用 commitAllowingStateLoss()，允许丢失一些界面的状态和信息。

#### 四、Fragment实现原理和Back Stack

我们知道 Activity 有任务栈，用户通过 startActivity 将 Activity 加入栈，点击返回按钮将 Activity 出栈。Fragment 也有类似的栈，称为回退栈（Back Stack），回退栈是由 FragmentManager 管理的。默认情况下，Fragment 事务是不会加入回退栈的，如果想将 Fragment 事务加入回退栈，则可以加入`addToBackStack("")`。如果没有加入回退栈，则用户点击返回按钮会直接将 Activity 出栈；如果加入了回退栈，则用户点击返回按钮会回滚 Fragment 事务。

我们将通过最常见的 Fragment 用法，讲解 Back Stack 的实现原理：

```java
getSupportFragmentManager().beginTransaction()
    .add(R.id.container, f1, "f1")
    .addToBackStack("")
    .commit();
```

上面这个代码的功能就是将 Fragment 加入 Activity 中，内部实现为：创建一个 BackStackRecord 对象，该对象记录了这个事务的全部操作轨迹（这里只做了一次 add 操作，并且加入回退栈），随后将该对象提交到 FragmentManager 的执行队列中，等待执行。

BackStackRecord 类的定义如下：

```java
//androidx之前
class BackStackRecord extends FragmentTransaction implements FragmentManager.BackStackEntry, Runnable {}

// androidx之后
class BackStackRecord extends FragmentTransaction implements
        FragmentManager.BackStackEntry, FragmentManagerImpl.OpGenerator {}
```

从定义可以看出，BackStackRecord 有三重含义：

- 继承了 FragmentTransaction，即是事务，保存了整个事务的全部操作轨迹。
- 实现了 BackStackEntry，作为回退栈的元素，正是因为该类拥有事务全部的操作轨迹，因此在popBackStack() 时能回退整个事务。
- 继承了 Runnable，即被放入 FragmentManager 执行队列，等待被执行。

先看第一层含义，`getSupportFragmentManager.beginTransaction()`返回的就是 BackStackRecord 对象，代码如下：

```java
public FragmentTransaction beginTransaction() {
        return new BackStackRecord(this);
    }
```

BackStackRecord 类包含了一次事务的整个操作轨迹，是以链表形式存在的，链表的元素是 Op 类，androidx 之后是保存在一个`ArrayList<Op> mOps = new ArrayList<>()`中，表示其中某个操作，定义如下：

```java
//androidx之前
static final class Op {
    Op next; //链表后一个节点
    Op prev; //链表前一个节点
    int cmd;  //操作是add或remove或replace或hide或show等
    Fragment fragment; //对哪个Fragment对象做操作
}
  
  // androidx之后
    static final class Op {
        int mCmd;
        Fragment mFragment;
        int mEnterAnim;
        int mExitAnim;
        int mPopEnterAnim;
        int mPopExitAnim;
        Lifecycle.State mOldMaxState;
        Lifecycle.State mCurrentMaxState;
    }
```

我们来看下具体场景下这些类是怎么被使用的，比如我们的事务做add操作。add 函数的定义：

```java
public FragmentTransaction add(int containerViewId, Fragment fragment, String tag) {
   doAddOp(containerViewId, fragment, tag, OP_ADD);
   return this;
}
```

`doAddOp()`方法就是创建 Op 对象，并加入链表，定义如下：

```java
private void doAddOp(int containerViewId, Fragment fragment, String tag, int opcmd) {
    fragment.mTag = tag;  //设置fragment的tag
    fragment.mContainerId = fragment.mFragmentId = containerViewId;  //设置fragment的容器id
    Op op = new Op();
    op.cmd = opcmd;
    op.fragment = fragment;
    addOp(op);
}
```

addOp() 是将创建好的 Op 对象加入链表，androidx 则加到列表，定义如下：

```java
void addOp(Op op) {
    if (mHead == null) {
        mHead = mTail = op;
    } else {
        op.prev = mTail;
        mTail.next = op;
        mTail = op;
    }
    mNumOp++;
}

//androidx
 void addOp(Op op) {
        mOps.add(op);
        op.mEnterAnim = mEnterAnim;
        op.mExitAnim = mExitAnim;
        op.mPopEnterAnim = mPopEnterAnim;
        op.mPopExitAnim = mPopExitAnim;
 }
```

`addToBackStack(“”)`是将 mAddToBackStack 变量记为 true，在 `commit()` 中会用到该变量。

```java
 public FragmentTransaction addToBackStack(@Nullable String name) {
        if (!mAllowAddToBackStack) {
            throw new IllegalStateException(
                    "This FragmentTransaction is not allowed to be added to the back stack.");
        }
        mAddToBackStack = true;
        mName = name;
        return this;
    }
```

`commit()`是异步的，即不是立即生效的，但是后面会看到整个过程还是在主线程完成，只是把事务的执行扔给主线程的 Handler，`commit()`内部是`commitInternal()`，实现如下：

```java
int commitInternal(boolean allowStateLoss) {
    mCommitted = true;
    if (mAddToBackStack) {
        mIndex = mManager.allocBackStackIndex(this);
    } else {
        mIndex = -1;
    }
    mManager.enqueueAction(this, allowStateLoss); //将事务添加进待执行队列中
    return mIndex;
}
```

如果 mAddToBackStack 为 true，则调用`allocBackStackIndex(this)`将事务添加进回退栈，FragmentManager 类的变量 mBackStackIndices 就是回退栈。实现如下：

```java
public int allocBackStackIndex(BackStackRecord bse) {
    if (mBackStackIndices == null) {
        mBackStackIndices = new ArrayList<BackStackRecord>();
    }
    int index = mBackStackIndices.size();
    mBackStackIndices.add(bse);
    return index;
}
```

在`commitInternal()`中，`mManager.enqueueAction(this, allowStateLoss);`是将 BackStackRecord 加入待执行队列中，定义如下：

```java
public void enqueueAction(Runnable action, boolean allowStateLoss) {
    if (mPendingActions == null) {
        mPendingActions = new ArrayList<Runnable>();
    }
    mPendingActions.add(action);
    if (mPendingActions.size() == 1) {
        mHost.getHandler().removeCallbacks(mExecCommit);
        mHost.getHandler().post(mExecCommit); //调用execPendingActions()执行待执行队列的事务
    }
}
```

mPendingActions 就是前面说的待执行队列，`mHost.getHandler()`就是主线程的Handler，因此 Runnable 是在主线程执行的，mExecCommit 的内部就是调用了`execPendingActions()`，即把 mPendingActions 中所有积压的没被执行的事务全部执行。执行队列中的事务会怎样被执行呢？就是调用`BackStackRecord的run()`方法，`run()`方法就是执行 Fragment 的生命周期函数，还有将视图添加进 container 中。

与`addToBackStack()`对应的是`popBackStack()`，有以下几种变种：

- popBackStack()：将回退栈的栈顶弹出，并回退该事务。
- popBackStack(String name, int flag)：name 为 addToBackStack(String name) 的参数，通过 name 能找到回退栈的特定元素，flag 可以为0或者 FragmentManager.POP_BACK_STACK_INCLUSIVE ，0表示只弹出该元素以上的所有元素，POP_BACK_STACK_INCLUSIVE 表示弹出包含该元素及以上的所有元素。这里说的弹出所有元素包含回退这些事务。
- popBackStack() 是异步执行的，是丢到主线程的 MessageQueue 执行，popBackStackImmediate() 是同步版本。

`getSupportFragmentManager().findFragmentByTag()`是经常用到的方法，是 FragmentManager 的方法，FragmentManager是抽象类，FragmentManagerImpl 是继承 FragmentManager 的实现类，他的内部实现是：

```java
class FragmentManagerImpl extends FragmentManager {
    ArrayList<Fragment> mActive;
    ArrayList<Fragment> mAdded;
    public Fragment findFragmentByTag(String tag) { 
           if (mAdded != null && tag != null) { 
               for (int i=mAdded.size()-1; i>=0; i--) {
                Fragment f = mAdded.get(i);
                if (f != null && tag.equals(f.mTag)) {
                        return f;
                }
            }
        }       
          if (mActive != null && tag != null) {
               for (int i=mActive.size()-1; i>=0; i--) {
                    Fragment f = mActive.get(i);
                    if (f != null && tag.equals(f.mTag)) {
                          return f;
                }
            }
        } 
          return null;
    }
}
```

从上面看到，先从 mAdded 中查找是否有该 Fragment，如果没找到，再从 mActive 中查找是否有该 Fragment。mAdded 是已经添加到 Activity 的 Fragment 的集合，mActive 不仅包含 mAdded，还包含虽然不在 Activity 中，但还在回退栈中的 Fragment。

#### 五、Fragment通信

##### Fragment向Activity传递数据

首先，在 Fragment 中定义接口，并让 Activity 实现该接口（具体实现省略）：

```java
public interface OnFragmentInteractionListener {   
  void onItemClick(String str);  //将str从Fragment传递给Activity
}
```

在 Fragment 的 onAttach() 中，将参数 Context 强转为 OnFragmentInteractionListener 对象：

```java
public void onAttach(Context context) {
    super.onAttach(context);
        if (context instanceof OnFragmentInteractionListener) {
        mListener = (OnFragmentInteractionListener) context;
    } else {
                throw new RuntimeException(context.toString()
                + " must implement OnFragmentInteractionListener");
    }
}
```

并在 Fragment 合适的地方调用`mListener.onItemClick("hello")`将”hello”从 Fragment 传递给 Activity。

##### Activity向Fragment传递数据

在 Fragment 初次创建时可通过 `fragment.setArguments(bundle)` 方法传递数据，其他时候，可以获取Fragment 对象，并调用 Fragment 的方法即可，比如要将一个字符串传递给 Fragment，则在 Fragment 中定义方法：

```java
public void setString(String str) { 
    this.str = str;
}
```

并在 Activity 中调用`fragment.setString("hello")`即可。

##### Fragment之间通信

由于 Fragment 之间是没有任何依赖关系的，因此如果要进行 Fragment 之间的通信，建议通过 Activity 作为中介，不要 Fragment 之间直接通信。

#### 六、Fragment中的onActivityResult

假设有一个 FragmentActivity 中嵌套一个 Fragment，它们各自使用 startActivityForResult 发起数据请求。
经测，目标所返回结果数据，能否被它们各自的 onActivityResult 方法所接收的情况如下：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghi3kbnbqwj30ks02tjrj.jpg)

- Fragment 和 FragmentActivity 都能接收到自己的发起的请求所返回的结果
- FragmentActivity 发起的请求，Fragment 完全接收不到结果
- Fragment 发起的请求，虽然在 FragmentActivity 中能获取到结果，但是requestCode完全对应不上

> Fragment.startActivityForResult
> ↓
> FragmentActivitymHost.HostCallbacks.onStartActivityFromFragment
> ↓
> FragmentActivity.startActivityFromFragment

从 Fragment 的 `startActivityForResult` 开始：

```java
 public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        if (mHost == null) {
            throw new IllegalStateException("Fragment " + this + " not attached to Activity");
        }
        mHost.onStartActivityFromFragment(this /*fragment*/, intent, requestCode, options);
    }
```

调用了一个mHost.onStartActivityFromFragment 的方法。Fragment 被添加到一个 FragmentActivity 中之后，这里的 mHost 即是当前 FragmentActivity 的一个内部类 FragmentActivity.HostCallbacks，它持有对FragmentActivity 的引用，通过调用 `onStartActivityFromFragment`：

```java
 @Override
        public void onStartActivityFromFragment(@NonNull Fragment fragment, Intent intent,
                int requestCode, @Nullable Bundle options) {
            FragmentActivity.this.startActivityFromFragment(fragment, intent, requestCode, options);
        }
```

转发到当前 FragmentActivity 的 `startActivityFromFragment`：

```java
public void startActivityFromFragment(Fragment fragment, Intent intent,
        int requestCode, @Nullable Bundle options) {
    mStartedActivityFromFragment = true;
    try {
        if (requestCode == -1) {
            ActivityCompat.startActivityForResult(this, intent, -1, options);
            return;
        }
        if ((requestCode&0xffff0000) != 0) {
            throw new IllegalArgumentException("Can only use lower 16 bits for requestCode");
        }
        int requestIndex = allocateRequestIndex(fragment);
        ActivityCompat.startActivityForResult(
            this, intent, ((requestIndex+1)<<16) + (requestCode&0xffff), options);
    } finally {
        mStartedActivityFromFragment = false;
    }
}
```

分析一下这段代码：
1.`mStartedActivityFromFragment = true`首先标记一下请求是来自于 Fragment。
2.`if(requestCode == -1)`的内容不用管，它是来自于 startActivity（没有ForResult）的情况。
3.然后的代码添加了对requestCode必须小于0xffff的限制 `if((requestCode&0xffff0000) ！= 0){/*抛异常*/}`
我们是从 Fragment.startActivityForResult 追踪到这里的，所以虽然文档没有明确说，但是从这里可以看出：**Fragment.startActivityForResult的requestCode也是必须要<=0xffff的。**

**然后，下面是关键点了：**

```java
ActivityCompat.startActivityForResult(
            this, intent, ((requestIndex+1)<<16) + (requestCode&0xffff), options);
```

其中ActivityCompat是一个帮助类，ActivityCompat.startActivityForResult 最终还是调用的Activity.startActivityForResult。通过分析，得知 requestIndex 是请求的序号，值为从0递增的整数值。
又从前面得知，requestCode 的本身的值是小于0xffff的，所以`((requestIndex+1)<<16)+(requestCode&0xffff)`简化一下就是：`(requestIndex+1)*65536+requestCode`——**所以这个值是必定大于0xffff的。**

再看一下 `FragmentActivity.startActivityForResult` 的代码：

```java
@Override
public void startActivityForResult(Intent intent, int requestCode) {
    // If this was started from a Fragment we've already checked the upper 16 bits were not in
    // use, and then repurposed them for the Fragment's index.
    if (!mStartedActivityFromFragment) {
        if (requestCode != -1 && (requestCode&0xffff0000) != 0) {
            throw new IllegalArgumentException("Can only use lower 16 bits for requestCode");
        }
    }
    super.startActivityForResult(intent, requestCode);
}
```

可以看到，判断了一下如果请求不是来自于 Fragment，也就是来自于 FragmentActivity 自身，就限制 requestCode 不能大于0xffff。

再加上前文所说的，Fragment.startActivityForResult 最终映射的 requestCode 值必定大于0xffff，所以，现在可以得出了一个初步的结果：
**SDK 把 Fragment 和 FragmentActivity 的 requestCode 都限制在了0xffff以内，然后对于 Fragment 所发起的请求，都通过一个映射，把最终的 requestCode 变成了一个大于0xffff的值。**

可以推测到：**在获取的结果的时候，也是会通过跟0xffff这个数值来比较，来区分是要把结果交给FragmentActivity 还是 Fragment 来处理。**

再来看一下 `FragmentActivity.onActivityResult` 方法

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    mFragments.noteStateNotSaved();
    int requestIndex = requestCode>>16;
    if (requestIndex != 0) {
        requestIndex--;
        String who = mPendingFragmentActivityResults.get(requestIndex);
        mPendingFragmentActivityResults.remove(requestIndex);
        if (who == null) {
            Log.w(TAG, "Activity result delivered for unknown Fragment.");
            return;
        }
        Fragment targetFragment = mFragments.findFragmentByWho(who);
        if (targetFragment == null) {
            Log.w(TAG, "Activity result no fragment exists for who: " + who);
        } else {
            targetFragment.onActivityResult(requestCode&0xffff, resultCode, data);
        }
        return;
    }
    super.onActivityResult(requestCode, resultCode, data);
}
```

果然，证实了我们上面的推论。在FragmentActivity.onActivityResult 中，只有 `requestCode>0xffff` 时，这里得到的 requestIndex 才能满足`requestIndex != 0`，然后进入下面的逻辑：把 requestCode 通过反向之前的映射关系，还原成最初 Fragment 所指定的 requestCode，交给 Fragment.onActivityResult 进行处理。

**注意：**通过 FragementActivity 源码可以发现，源码里没有处理嵌套 Fragment 的情况，也就是说回调只到第一级Fragment，就没有继续分发。所以在第二级或者更深级别的 Fragment 调用 startActivityForResult 方法时，将无法收到 onActivityResult 回调。

- **使用 startActivityForResult 的时候，requestCode 一定不要大于 0xffff(65535)**。
- 嵌套一层 Fragment 时，要在 Fragment 的 onActivityResult 接收数据，在 Fragment 中要使用 `Fragment.startActivityForResult`，而不是 `Fragment.getActivity().startActivityForResult`，如果 activity 中重写了 onActivityResult，那么一定要加上`super.onActivityResult(requestCode, resultCode, data)`。

- 嵌套多层 Fragment 时，要在第二级或更深级别的 Fragment 获取回调，需要重写 activity 的 onActivityResult 方法，继续分发回调给 Fragment。

```java
public class CustomAppCompatActivity extends AppCompatActivity {
    private static final String TAG = "TAG";

    /**
     * 重写onactivityresult方法，使二个或多个fragment嵌套使用时能收到onactivityresut回调
     * @param requestCode
     * @param resultCode
     * @param data
     */
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        FragmentManager fm = getSupportFragmentManager();
        int index = requestCode >> 16;
        if (index != 0) {
            index--;
            if (fm.getFragments() == null || index < 0
                    || index >= fm.getFragments().size()) {
                Log.w(TAG, "Activity result fragment index out of range: 0x"
                        + Integer.toHexString(requestCode));
                return;
            }
            Fragment frag = fm.getFragments().get(index);
            if (frag == null) {
                Log.w(TAG, "Activity result no fragment exists for index: 0x"
                        + Integer.toHexString(requestCode));
            } else {
                handleResult(frag, requestCode, resultCode, data);
            }
            return;
        }

    }

    /**
     * 递归调用，对所有子Fragement生效
     *
     * @param frag
     * @param requestCode
     * @param resultCode
     * @param data
     */
    private void handleResult(Fragment frag, int requestCode, int resultCode,
                              Intent data) {
        frag.onActivityResult(requestCode & 0xffff, resultCode, data);
        List<Fragment> frags = frag.getChildFragmentManager().getFragments();
        if (frags != null) {
            for (Fragment f : frags) {
                if (f != null)
                    handleResult(f, requestCode, resultCode, data);
            }
        }
    }
}
```

在 Fragment 中调用 startActivityForResult 时，一定要调用根 Fragment 的启动方法，如下：

```java
/**
  * 得到根Fragment
  * 
  * @return
  */
 private Fragment getRootFragment() {
  Fragment fragment = getParentFragment();
  while (fragment.getParentFragment() != null) {
   fragment = fragment.getParentFragment();
  }
  return fragment;

 }

 /**
  * 启动Activity
  */
 private void onClickTextViewRemindAdvancetime() {
  Intent intent = new Intent();
  intent.setClass(getActivity(), YourActivity.class);
  intent.putExtra("TAG","TEST"); 
  getRootFragment().startActivityForResult(intent, 1000);
 }
```

#### 七、ViewPager+Fragment相关

ViewPager 是 android 中提供界面滑动的类，继承自 ViewGroup。PagerAdapter 是 ViewPager 的适配器类，为ViewPager提供界面。但是一般来说，通常都会使用 PagerAdapter 的两个子类：FragmentPagerAdapter和FragmentStatePagerAdapter 作为 ViewPager 的适配器，他们的特点是界面是 Fragment 。

ViewPager 默认会缓存当前页相邻的`DEFAULT_OFFSCREEN_PAGES`(默认1)个界面，比如当滑动到第2页时，会初始化第1页和第3页的界面（即 Fragmen t对象，且生命周期函数运行到`onResume()`），可以通过`setOffscreenPageLimit(count)`设置当前页的左右两边的预加载界面数量。

FragmentPagerAdapter 和 FragmentStatePagerAdapter 需要重写的方法都一样，常见的重写方法如下：

- public FragmentPagerAdapter(FragmentManager fm)：构造函数，参数为 FragmentManager。如果是嵌套 Fragment 场景，子 PagerAdapter的参数传入 getChildFragmentManager()。
- Fragment getItem(int position)：返回第 position 位置的 Fragment，必须重写。
- int getCount(): 返回 ViewPager 的页数，必须重写。
- Object instantiateItem(ViewGroup container, int position)：container 是 ViewPager 对象，返回第 position位置的 Fragment。
- void destroyItem(ViewGroup container, int position, Object object)：container 是 ViewPager 对象，object 是 Fragment 对象。
- getItemPosition(Object object)：object 是 Fragment 对象，如果返回 POSITION_UNCHANGED，则表示当前 Fragment 不刷新，如果返回 POSITION_NONE，则表示当前 Fragment 需要调用`destroyItem()`和`instantiateItem()`进行销毁和重建。 默认情况下返回 POSITION_UNCHANGED。

FragmentPagerAdapter 和 FragmentStatePagerAdapter 的区别：

- **FragmentPagerAdapter**：**对Fragment的状态没有恢复和保存，对Fragment对象进行视图销毁**。每个Fragment 会持久的保存在 FragmentManager 中，在 destroyItem() 中只是 detach，只是在页面上让 fragment 的UI脱离 Activity，但 fragment 仍然保存在内存里，并不会回收内存。因此适用于那些数据**相对静态**的页，Fragment **数量也比较少**的情况。
- **FragmentStatePagerAdapter**： **对Fragment的状态进行了恢复和保存，对Fragment对象进行实例销毁** 。只保留当前页面，当页面不可见时，在 destroyItem() 中会 remove 之前加载的 fragment，该 fragment 就会被消除，释放其内存资源。因此适用于那些**数据动态性**较大、**占用内存**较多，Fragment **数量较多**的情况。

#### 八、懒加载

##### 为什么要使用懒加载

默认情况，ViewPager 会缓存当前页和左右相邻的界面。实现懒加载的主要原因是：用户没进入的界面需要有一系列的网络、数据库等耗资源、耗时的操作，预先做这些数据加载是不必要的。为了不做预先额外的数据加载，节省资源，就需要使用懒加载。

##### 什么是懒加载

当界面对用户可见时，才加载数据并更新UI，当界面对用户不可见时，停止加载数据等一切操作。

##### 如何实现懒加载

**在 androidx 之前**，懒加载主要依赖 Fragment 的`setUserVisibleHint(boolean isVisible)`方法，当Fragment变为可见时，会调用`setUserVisibleHint(true)`；当Fragment变为不可见时，会调用`setUserVisibleHint(false)`，且该方法调用时机：

- onAttach() 之前，调用`setUserVisibleHint(false)`。
- onCreateView() 之前，如果该界面为当前页，则调用`setUserVisibleHint(true)`，否则调用`setUserVisibleHint(false)`。
- 界面变为可见时，调用`setUserVisibleHint(true)`。
- 界面变为不可见时，调用`setUserVisibleHint(false)`。

懒加载 Fragment 的实现：

```java
public class LazyFragment extends Fragment {    
    private View mRootView;
    private boolean mIsPrepared; //表示UI是否准备好
    private boolean mIsInited;  //表示是否已经做过数据加载
                                            
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        mRootView = inflater.inflate(R.layout.fragment_lazy, container, false);
        mIsPrepared = true;
        lazyLoad();// 解决第一次数据显示
        return mRootView;
    }
            
    public void lazyLoad() {
       if (getUserVisibleHint() && mIsPrepared && !mIsInited) { 
           //异步初始化，在初始化后显示正常UI
           //1. 加载数据  2. 更新UI  3. mIsInited = true
           loadData();
           mIsInited = true
        }
    }
     
    //子类可重写
    public void loadData() {
        // 加载数据
    }   
   
    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) { 
           super.setUserVisibleHint(isVisibleToUser);
           if (isVisibleToUser) {
             lazyLoad();
           }
    }
   
    public static LazyFragment newInstance() {
           return new LazyFragment();
    }
}
```

- mIsPrepared：表示UI是否准备好，因为数据加载后需要更新UI，如果UI还没有inflate，就不需要做数据加载，因为setUserVisibleHint()会在onCreateView()之前调用一次，如果此时调用，UI还没有inflate，因此不能加载数据。
- mIsInited：表示是否已经做过数据加载，如果做过了就不需要做了。因为 setUserVisibleHint(true) 在界面可见时都会调用，如果滑到该界面做过数据加载后，滑走，再滑回来，还是会调用 setUserVisibleHint(true)，此时由于mIsInited=true，因此不会再做一遍数据加载。
- lazyLoad()：懒加载的核心类，在该方法中，只有界面可见（getUserVisibleHint()==true）、UI准备好（mIsPrepared==true）、过去没做过数据加载（mIsInited==false）时，才需要调loadData()做数据加载，数据加载做完后把mIsInited置为true。

**在 androidx 之后**，`setUserVisibleHint()`方法已经过时了，官方提出了`setMaxLifecycle()`方法来替代`setUserVisibleHint()`方法。

**setMaxLifecycle()方法**

```java
/**
     * Set a ceiling for the state of an active fragment in this FragmentManager. If fragment is
     * already above the received state, it will be forced down to the correct state.
     *
     * <p>The fragment provided must currently be added to the FragmentManager to have it's
     * Lifecycle state capped, or previously added as part of this transaction. The
     * {@link Lifecycle.State} passed in must at least be {@link Lifecycle.State#CREATED}, otherwise
     * an {@link IllegalArgumentException} will be thrown.</p>
     *
     * @param fragment the fragment to have it's state capped.
     * @param state the ceiling state for the fragment.
     * @return the same FragmentTransaction instance
     */
@NonNull
public FragmentTransaction setMaxLifecycle(@NonNull Fragment fragment,
            @NonNull Lifecycle.State state) {
  addOp(new Op(OP_SET_MAX_LIFECYCLE, fragment, state));
  return this;
}
```

`setMaxLifecycle()`方法定义在**FragmentTransaction**类中，它的内部逻辑很简单，其实我们经常使用的`add()`、`remove()`、`show()`、`hide()`等方法也是类似的逻辑，将操作封装为一个Op对象，最后调用`commit()`方法时再根据Op对象执行对应的操作。

注释中提到`setMaxLifecycle()`方法的作用是为 Fragment 的状态设置上限，如果当前 Fragment 的状态已经超过了设置的上限，就会强制被降到相应状态。在弄清楚上面这段文字的意义之前我首先要介绍两个相关概念：**Fragment的状态**和**Lifecycle的状态**。

- **Fragment的状态**

在Fragment类中定义了5个int常量，表示Fragment的状态值：

```java
static final int INITIALIZING = 0;     // Not yet created.
static final int CREATED = 1;          // Created.
static final int ACTIVITY_CREATED = 2; // Fully created, not started.
static final int STARTED = 3;          // Created and started, not resumed.
static final int RESUMED = 4;          // Created started and resumed.
```

- **Lifecycle的状态**

[Lifecycle](https://developer.android.google.cn/topic/libraries/architecture/lifecycle)是 Android Jetpack 中的架构组件之一，用于帮助我们方便地管理 Activity 和 Fragment 的生命周期，关于 Lifecycle 的详细介绍和使用网上有很多文章，我这里就不说了，如果此前没有接触过可以自行了解一下哈。
在 Lifecycle 定义了一个枚举类**State**：

```java
public enum State {
    DESTROYED,
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED;
    public boolean isAtLeast(@NonNull State state) {
        return compareTo(state) >= 0;
    }
}
```

可以看出 Lifecycle 中同样定义了5个状态，不过这里的状态和 Fragment中 定义的状态还是有一些区别的。

回到`setMaxLifecycle()`方法，需要传入的参数有两个：fragment和state。fragment不用多说，就是要设置的目标Fragment，不过需要注意的是**此时Fragment必须已经被添加到了FragmentManager中，也就是调用了`add()`方法**，否则会抛出异常。state就是Lifecycle中定义的枚举类型，同样需要注意**传入的state应该至少为CREATED，换句话说就是只能传入CREATED、STARTED和RESUMED**，否则同样会抛出异常。

下面就以我们最熟悉的生命周期方法来说明这个状态的限制，先上一张图总结一下结论：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghkm2rcvpcj32bg0u0tn8.jpg)

图中展示了 Fragment 状态间切换会执行的生命周期方法以及 Lifecycle.State 对应的 Fragment 状态，由于`setMaxLifecycle()`方法要求传入的state至少为**CREATED**，因此我们只需研究**CREATED**、**STARTED**和**RESUMED**这三个状态，结合上图解释一下`setMaxLifecycle()`方法的作用。

- **参数传入Lifecycle.State.CREATED**

**Lifecycle.State.CREATED**对应Fragment的**CREATED**状态，如果当前Fragment状态低于**CREATED**，也就是**INITIALIZING**，那么Fragment的状态会变为**CREATED**，依次执行`onAttach()`、`onCreate()`方法；如果当前Fragment状态高于**CREATED**，那么Fragment的状态会被强制降为**CREATED**，以当前Fragment状态为**RESUMED**为例，接下来会依次执行`onPause()`、`onStop()`和`onDestoryView()`方法。如果当前Fragment的状态恰好为**CREATED**，那么就什么都不做。

- **参数传入Lifecycle.State.STARTED**

**Lifecycle.State.STARTED**对应Fragment的**STARTED**状态，如果当前Fragment状态低于**STARTED**，那么Fragment的状态会变为**STARTED**，以当前Fragment状态为**CREATED**为例，接下来会依次执行`onCreateView()`、`onActivityCreate()`和`onStart()`方法；如果当前Fragment状态高于**STARTED**，也就是**RESUMED**，那么Fragment的状态会被强制降为**STARTED**，接下来会执行`onPause()`方法。如果当前Fragment的状态恰好为**STARTED**，那么就什么都不做。

- **参数传入Lifecycle.State.RESUMED**

**Lifecycle.State.RESUMED**对应Fragment的**RESUMED**状态，如果当前Fragment状态低于**RESUMED**，那么Fragment的状态会变为**RESUMED**，以当前Fragment状态为**STARTED**为例，接下来会执行`onResume()`方法。如果当前Fragment的状态恰好为**RESUMED**，那么就什么都不做。

那么 androidx 之后懒加载的新方案，这次的切入点在 **FragmentPagerAdapter** 中，我们会发现之前继承自 FragmentPagerAdapter 的构造方法同样过时了。提供了新的构造方法：

```java
public FragmentPagerAdapter(@NonNull FragmentManager fm,
                            @Behavior int behavior) {
    mFragmentManager = fm;
    mBehavior = behavior;
}
```

多了一个 int 类型的参数 behavior，可选的值有以下两个：

```java
public static final int BEHAVIOR_SET_USER_VISIBLE_HINT = 0;

public static final int BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT = 1;
```

可以看出一个参数的构造方法默认传入**BEHAVIOR_SET_USER_VISIBLE_HINT**，将其赋值给mBehavior，那么这个mBehavior在什么地方用到了呢。在**FragmentPagerAdapter.java**文件中全局搜索一下，发现只有两个地方用到了mBehavior：`instantiateItem()`方法和`setPrimaryItem()`方法。`instantiateItem()`方法我们很熟悉，是初始化ViewPager中每个Item的方法，`setPrimaryItem()`方法我此前没有接触过，简单地看了一下源码发现它的作用是设置ViewPager将要显示的Item，在ViewPager切换时会调用该方法，我们来看一下FragmentPagerAdapter中的`setPrimaryItem()`方法：

```java
public void setPrimaryItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
        Fragment fragment = (Fragment)object;
        if (fragment != mCurrentPrimaryItem) {
            if (mCurrentPrimaryItem != null) {
                mCurrentPrimaryItem.setMenuVisibility(false);
                if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
                    if (mCurTransaction == null) {
                        mCurTransaction = mFragmentManager.beginTransaction();
                    }
                    mCurTransaction.setMaxLifecycle(mCurrentPrimaryItem, Lifecycle.State.STARTED);
                } else {
                    mCurrentPrimaryItem.setUserVisibleHint(false);
                }
            }
            fragment.setMenuVisibility(true);
            if (mBehavior == BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT) {
                if (mCurTransaction == null) {
                    mCurTransaction = mFragmentManager.beginTransaction();
                }
                mCurTransaction.setMaxLifecycle(fragment, Lifecycle.State.RESUMED);
            } else {
                fragment.setUserVisibleHint(true);
            }
            mCurrentPrimaryItem = fragment;
        }
    }
```

如果 mBehavior 的值为**BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT**，那么就调用`setMaxLifecycle()`方法将上一个Fragment的状态设置为**STARTED**，将当前要显示的Fragment的状态设置为**RESUMED**；反之如果mBehavior的值为**BEHAVIOR_SET_USER_VISIBLE_HINT**，那么依然使用`setUserVisibleHint()`方法设置Fragment的可见性，相应地可以根据`getUserVisibleHint()`方法获取到Fragment是否可见，从而实现懒加载，具体做法：

- 在构造 Adapter 对象的时候 behavior 参数传入 **FragmentPagerAdapter.BEHAVIOR_RESUME_ONLY_CURRENT_FRAGMENT**；
- 将 Fragment 加载数据的逻辑放到 onResume() 方法中，这样就保证了 Fragment 可见时才会加载数据。

- 声明一个变量标记是否是首次执行`onResume()`方法，因为每次 Fragment 由不可见变为可见都会执行`onResume()`方法，需要防止数据的重复加载。此外，如果我们使用的是 FragmentPagerAdapter，切换导致Fragment被销毁时是不会执行`onDestory()`和`onDetach()`方法的，只会执行到`onDestroyView()`方法，因此在`onDestroyView()`方法中我们还需要将这个变量重置，否则当Fragment再次可见时就不会重新加载数据了。

以上几点我们就可以封装出新的懒加载 Fragment 了，完整代码如下：

```java
public abstract class NewLazyFragment extends Fragment {
    private Context mContext;
    private boolean isFirstLoad = true; // 是否第一次加载

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mContext = getActivity();
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(getLayoutRes(),container,false);
        initView(view);
        return view;
    }
    
    @Override
    public void onDestroyView() {
        super.onDestroyView();
        isFirstLoad = true;
    }    
    
    @Override
    public void onResume() {
        super.onResume();
        if (isFirstLoad) {
            // 将数据加载逻辑放到onResume()方法中
            initData();
            isFirstLoad = false;
        }
    }

    /**
     * 设置布局资源id
     *
     * @return
     */
    protected abstract int getLayoutRes();

    /**
     * 初始化视图
     *
     * @param view
     */
    protected void initView(View view) {

    }

    /**
     * 初始化数据
     */
    protected void initData() {

    }
}
```

### 总结

本文记录 Android 中 Fragment 的相关知识点，包括 Fragment 的基本定义及使用、生命周期、回退栈的内部实现、Fragment 通信、ViewPager+Fragment 的使用、AndroidX前、后的懒加载等，加深了对 Fragment 的理解和学习。

[Demo](https://github.com/prsuit/Android-Learn-Sample)

引用文章：

> [Android基础：Fragment，看这篇就够了](https://mp.weixin.qq.com/s/dUuGSVhWinAnN9uMiBaXgw)
>
> [彻底搞懂startActivityForResult在FragmentActivity和Fragment中的异同](https://www.jianshu.com/p/ca91fa528d5c)
>
> [Android的Fragment中onActivityResult不被调用的解决方案](https://www.cnblogs.com/xjx22/p/5263658.html)
>
> [androidx中的Fragment懒加载方案](https://blog.csdn.net/qq_36486247/article/details/102531304)

