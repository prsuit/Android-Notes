---
title: ThreadLocal源码分析
categories:
  - Android基础
tags:
  - Android基础
date: 2020-08-08 21:38:54
---

### 前言

本文是介绍Android中的ThreadLocal。

### 目录

#### 一、ThreadLocal是什么

ThreadLocal 是线程局部(本地)变量，也许把它命名为 ThreadLocalVariable 更容易让人理解一些。官方介绍：

> This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its `get` or `set` method) has its own, independently initialized copy of the variable. `ThreadLocal` instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).

<!--more-->

ThreadLocal 提供了线程内存储变量的能力，这些变量不同之处在于每一个线程读取的变量是对应的互相独立的。通过 get 和 set 方法就可以得到当前线程对应的值。

**产生背景**：在 Java 的多线程编程中，为保证多个线程对共享变量的安全访问，通常会使用 synchronized 来保证同一时刻只有一个线程对共享变量进行操作。当**每个线程需要一个独享的对象**或者**线程内要保存全局变量**时，可以将变量放到 ThreadLocal 类型的对象中，使变量在每个线程中都有独立拷贝，不会出现一个线程读取变量时而被另一个线程修改的现象。最常见的 ThreadLocal 使用场景为用来解决数据库连接、Session 管理等。

**本质**：Thread 类有一个 ThreadLocal.ThreadLocalMap 类型的实例变量 threadLocals，ThreadLocal 的静态内部类 ThreadLocalMap 为每个 Thread 都维护了一个数组 table，ThreadLocal 对象确定了一个数组下标，而这个下标就是 value 存储的对应位置。

**作用**：为解决线程安全问题一个很好的思路，它通过为每个线程提供一个独立的变量副本解决了变量并发访问的冲突问题。在很多情况下，ThreadLocal 比直接使用 synchronized 同步机制解决线程安全问题更简单，更方便，且结果程序拥有更高的并发性。

#### 二、使用

##### 创建ThreadLocal变量

```java
// 1. 直接创建对象
private ThreadLocal myThreadLocal = new ThreadLocal()

// 2. 创建泛型对象
private ThreadLocal myThreadLocal = new ThreadLocal<String>();

// 3. 创建泛型对象 & 初始化值
// 指定泛型的好处：不需要每次对使用get()方法返回的值作强制类型转换
private ThreadLocal myThreadLocal = new ThreadLocal<String>() {
    @Override
    protected String initialValue() {
        return "This is the initial value";
    }
};

// 特别注意：
// 1. ThreadLocal实例 = 类中的private、static字段
// 2. 只需实例化对象一次 & 不需知道它是被哪个线程实例化
// 3. 每个线程都保持 对其线程局部变量副本 的隐式引用
// 4. 线程消失后，其线程局部实例的所有副本都会被垃圾回收（除非存在对这些副本的其他引用）
// 5. 虽然所有的线程都能访问到这个ThreadLocal实例，但是每个线程只能访问到自己通过调用ThreadLocal的set（）设置的值
 // 即 哪怕2个不同的线程在同一个ThreadLocal对象上设置了不同的值，他们仍然无法访问到对方的值
```

##### 访问ThreadLocal变量

```java
// 1. 设置值：set()
// 需要传入一个Object类型的参数
myThreadLocal.set("初始值”);

// 2. 读取ThreadLocal变量中的值：get()
// 返回一个Object对象
String threadLocalValue = (String) myThreadLocal.get();
```

#### 三、源码分析

##### set

```java
//set 方法
public void set(T value) {
      //获取当前线程
      Thread t = Thread.currentThread();
      //获取该线程的ThreadLocalMap对象
      ThreadLocalMap map = getMap(t);
      //若该线程的ThreadLocalMap对象已存在，则设置该Map里ThreadLocal的值；否则创建1个ThreadLocalMap对象
      if (map != null)
          map.set(this, value);
      else
          createMap(t, value);
  }
  
//getMap方法，获取当前线程的threadLocals变量引用
ThreadLocalMap getMap(Thread t) {
      //thred中维护了一个ThreadLocalMap
      return t.threadLocals;
 }
 
//createMap方法，创建当前线程的ThreadLocalMap对象
void createMap(Thread t, T firstValue) {
      //实例化一个新的ThreadLocalMap，并赋值给线程的成员变量threadLocals
      // a. ThreadLocalMap的键Key = 当前ThreadLocal实例
      // b. ThreadLocalMap的值 = 该线程设置的存储在ThreadLocal变量的值
      t.threadLocals = new ThreadLocalMap(this, firstValue);
}

// Thread类 源码分析
public class Thread implements Runnable {
  ...
    
  /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
     // 即 Thread类持有threadLocals变量
     // 线程类实例化后，每个线程对象拥有独立的threadLocals变量变量
     // threadLocals变量在 ThreadLocal对象中 通过set（） 或 get（）进行操作
     ThreadLocal.ThreadLocalMap threadLocals = null;
  ...
}
```

可以看出**每个线程持有一个ThreadLocalMap对象**。每一个新的线程Thread都会实例化一个ThreadLocalMap并赋值给成员变量 threadLocals，使用时若已经存在threadLocals则直接使用已经存在的对象。

##### ThreadLocalMap

```java
static class ThreadLocalMap {
  //Entry为ThreadLocalMap静态内部类，对ThreadLocal的弱引用
  //同时让ThreadLocal和储值形成key-value的关系
  static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
           super(k);
            value = v;
    }
}
  //数组的初始容量大小
  private static final int INITIAL_CAPACITY = 16;
  //存放数据的Entry类型数组
  private Entry[] table;

 //ThreadLocalMap构造方法
 ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        //内部成员数组，INITIAL_CAPACITY值为16的常量
        table = new Entry[INITIAL_CAPACITY];
        //位运算，结果与取模相同，计算出需要存放的位置
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
}
```

实例化 ThreadLocalMap 时创建了一个长度为16的 Entry 数组。通过 hashCode 与 length 位运算确定出一个索引值i，这个i就是被存储在 table 数组中的位置。每个线程 Thread 持有一个 ThreadLocalMap 类型的实例threadLocals 可理解成每个线程 Thread 都持有一个 Entry 型的数组 table，而一切的读取过程都是通过操作这个数组 table 完成的。

##### ThreadLocalMap的set

```java
private void set(ThreadLocal<?> key, Object value) {
  Entry[] tab = table;
  int len = tab.length;
  //获取索引值，
  int i = key.threadLocalHashCode & (len-1);

  //遍历tab如果已经存在则更新值
  for (Entry e = tab[i];e != null;e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();
    if (k == key) {
      e.value = value;
      return;
    }
    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }
  //如果上面没有遍历成功则创建新值
  tab[i] = new Entry(key, value);
  int sz = ++size;
  //满足条件数组扩容x2
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}
```

在 ThreadLocalMap 中的 set 方法与构造方法能看到以下代码片段。

- `int i = key.threadLocalHashCode & (len-1)`
- `int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1)`
  简而言之就是将threadLocalHashCode进行一个位运算（取模）得到索引i，threadLocalHashCode代码如下：

```java
    //ThreadLocal中threadLocalHashCode相关代码.
    private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        //自增
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```

因为static的原因，在每次`new ThreadLocal`时因为 threadLocalHashCode 的初始化，会使threadLocalHashCode 值自增一次，增量为0x61c88647。

0x61c88647是斐波那契散列乘数,它的优点是通过它散列(hash)出来的结果分布会比较均匀，可以很大程度上避免hash冲突。

综上可以得出：

1.对于某一 ThreadLocal 来讲，他的索引值i是确定的，在不同线程之间访问时访问的是不同的 table 数组的同一位置即都为 table[i]，只不过这个不同线程之间的 table 是独立的。

2.对于同一线程的不同 ThreadLocal 来讲，这些 ThreadLocal 实例共享一个 table 数组，然后每个 ThreadLocal 实例在 table 中的索引 i 是不同的。

即 ThreadLocal 中 set 和 get 操作的都是对应线程的 table数组，因此在不同的线程中访问同一个 ThreadLocal 对象的 set 和 get 进行存取数据是不会相互干扰的。

##### get

通过计算出索引直接从数组对应位置读取即可。

```java
//ThreadLocal中get方法
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    //若该线程的ThreadLocalMap对象已存在，则直接获取该Map里的值；
    //否则通过初始化函数创建1个ThreadLocalMap对象
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
    
//ThreadLocalMap中getEntry方法
private Entry getEntry(ThreadLocal<?> key) {
       int i = key.threadLocalHashCode & (table.length - 1);
       Entry e = table[i];
       if (e != null && e.get() == key)
            return e;
       else
            return getEntryAfterMiss(key, i, e);
   }
```

##### setInitialValue

```java
/** 
    * 初始化ThreadLocal的值
    **/ 
    private T setInitialValue() {

        T value = initialValue();

        // 1. 获得当前线程
        Thread t = Thread.currentThread();

        // 2. 获取该线程的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);

         // 3. 若该线程的ThreadLocalMap对象已存在，则直接替换该值；否则则创建
        if (map != null)
            map.set(this, value); // 替换
        else
            createMap(t, value); // 创建
        return value;
    }
```

##### initialValue

重写该方法可初始化值，默认情况下，initialValue方法返回的是null。

```java
protected T initialValue() {
        return null;
    }
```

##### remove

```java
//ThreadLocal中remove方法
public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
}

 			/**
         * Remove the entry for key.
         */
      //ThreadLocalMap中remove方法
       private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();//这里的clear()方法实际上Reference中提供的方法
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```

##### ThreadLocal如何做到线程安全

- 每个线程拥有自己独立的`ThreadLocals`变量（指向`ThreadLocalMap`对象 ）
- 每当线程 访问 `ThreadLocals`变量时，访问的都是各自线程自己的`ThreadLocalMap`变量（键 - 值）
- `ThreadLocalMap`变量的键 `key` = 唯一 = 当前`ThreadLocal`实例

##### 与同步机制的区别

ThreadLocal 和 Synchronized 都是为了解决多线程中相同变量的访问冲突问题，不同的点是

- Synchronized 是通过线程等待，牺牲时间来解决访问冲突，以时间换空间
- ThreadLocal 是通过每个线程单独一份存储空间，牺牲空间来解决冲突，以空间换时间，并且相比于Synchronized，ThreadLocal 具有线程隔离的效果，只有在线程内才能获取到对应的值，线程外则不能访问到想要的值。

因为 ThreadLocal 的线程隔离特性，在 android 中Looper、ActivityThread 以及 AMS 中都用到了 ThreadLocal。当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用 ThreadLocal。

##### 防止ThreadLocal中的内存泄漏

`ThreadLocalMap`的键`Key` = 当前`ThreadLocal`实例、值`value` = 该线程设置的存储在`ThreadLocal`变量的值

该`key`是 `ThreadLocal`对象的弱引用；当要抛弃掉`ThreadLocal`对象时，垃圾收集器会忽略该 `key`的引用而清理掉`ThreadLocal`对象

**ThreadLocalMap底层会维护一个Entry数组，而Entry本身却是WeakReference的子类，并且在构造器中将ThreadLocal传给了父类WeakReference。也就是说，Entry对于ThreadLocal持有的引用是弱引用，它并不会影响GC对于ThreadLocal对象的回收**。

但是对于value 依旧是强引用，如果一个ThreadLocal对象被回收了，我们往里面放的value对于**【当前线程->当前线程的threadLocals(ThreadLocal.ThreadLocalMap对象）->Entry数组->某个entry.value】**这样一条强引用链是可达的，因此value不会被回收。如果不及时清理释放，是会导致内存泄漏的。所以，我们在不使用时，最好调用 ThreadLocal 的 `remove()` 方法。

### 总结

在每个线程 Thread 内部有一个 ThreadLocal.ThreadLocalMap 类型的成员变量 threadLocals，这个threadLocals 就是用来存储实际的变量副本的，键值为当前 ThreadLocal 变量，value 为变量副本值(即T类型对象值)。

初始时，在 Thread 里面，threadLocals 为空，当通过 ThreadLocal 变量调用 get() 方法或者 set() 方法，就会对Thread类中的 threadLocals 进行初始化，并且以当前 ThreadLocal 变量为键值，以 ThreadLocal 要保存的副本变量为 value，存到 threadLocals。 然后在当前线程里面，如果要使用副本变量，就可以通过 get 方法在threadLocals 里面查找。

- 每个Thread维护着一个ThreadLocalMap的引用；
- ThreadLocalMap是ThreadLocal的内部类，用Entry来进行存储；
- ThreadLocal创建的副本是存储在线程自己的threadLocals中的，也就是线程自己的ThreadLocalMap；
- ThreadLocalMap的键值为ThreadLocal对象，因为每个线程可以有多个threadLocal变量，因此保存在ThreadLocalMap中，键值为ThreadLocal对象；
- ThreadLocal本身并不存储值，它只是作为一个key来让线程从ThreadLocalMap获取value。

引用文章：

> [Java多线程：带你了解神秘的线程变量 ThreadLocal](https://www.jianshu.com/p/22be9653df3f)
>
> [ThreadLocal](https://www.jianshu.com/p/3c5d7f09dfbd)
>
> [ThreadLocal源码解读](https://www.cnblogs.com/micrari/p/6790229.html)

