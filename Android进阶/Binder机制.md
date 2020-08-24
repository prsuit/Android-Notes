---
title: Binder机制
categories:
  - Android进阶
tags:
  - Android进阶
date: 2020-08-12 22:42:35
---

### 前言

本文是介绍Android中的Binder机制。

### 目录

#### 一、Binder是什么

Binder 是 Android 系统进程间的一种通信机制，官方介绍：

> Base class for a remotable object, the core part of a lightweight remote procedure call mechanism defined by `IBinder`. This class is an implementation of IBinder that provides standard local implementation of such an object.

远程对象的基类，它是 IBinder 定义的轻量级远程过程调用机制的核心部分。此类是 IBinder 的实现，它提供此类本地对象的标准实现。

<!--more-->

我们知道 Android 应用程序是由 Activity、Service、Broadcast Receiver 和 Content Provider 四大组件中的一个或者多个组成的。有时这些组件运行在同一进程，有时运行在不同的进程。这些进程间的通信就依赖于 Binder IPC 机制。不仅如此，Android 系统对应用层提供的各种服务如：ActivityManagerService、PackageManagerService 等都是基于 Binder IPC 机制来实现的。Binder 机制在 Android 中的位置非常重要。

####  二、为什么要用Binder 

Android 系统是基于 Linux 内核的，Linux 已经提供了管道、消息队列、共享内存和 Socket 等 IPC 机制。那为什么 Android 还要提供 Binder 来实现 IPC 呢？主要是基于**性能**、**稳定性**和**安全性**几方面的原因。

##### 性能

首先说说性能上的优势。Socket 作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。**Binder 只需要一次数据拷贝**，性能上仅次于共享内存。注：各种IPC方式数据拷贝次数，此表来源于[Android Binder 设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)

| IPC方式            | **数据拷贝次数** |
| ------------------ | ---------------- |
| 共享内存           | 0                |
| Binder             | 1                |
| Socket/管道/消息队 | 2                |

##### 稳定性

再说说稳定性，**Binder 基于 C/S 架构**，客户端（Client）有什么需求就丢给服务端（Server）去完成，架构清晰、职责明确又相互独立，自然稳定性更好。共享内存虽然无需拷贝，但是控制负责，难以使用。从稳定性的角度讲，Binder 机制是优于内存共享的。

##### 安全性

另一方面就是安全性。Android 作为一个开放性的平台，市场上有各类海量的应用供用户选择安装，因此安全性对于 Android 平台而言极其重要。作为用户当然不希望我们下载的 APP 偷偷读取我的通信录，上传我的隐私数据，后台偷跑流量、消耗手机电量。传统的 IPC 没有任何安全措施，完全依赖上层协议来确保。首先传统的 IPC 接收方无法获得对方可靠的进程用户ID/进程ID（UID/PID），从而无法鉴别对方身份。**Android 为每个安装好的 APP 分配了自己的 UID，故而进程的 UID 是鉴别进程身份的重要标志**。传统的 IPC 只能由用户在数据包中填入 UID/PID，但这样不可靠，容易被恶意程序利用。可靠的身份标识只有由 IPC 机制在内核中添加。其次传统的 IPC 访问接入点是开放的，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法阻止恶意程序通过猜测接收方地址获得连接。同时 **Binder 既支持实名 Binder，又支持匿名 Binder**，安全性高。

基于上述原因，Android 需要建立一套新的 IPC 机制来满足系统对稳定性、传输性能和安全性方面的要求，这就是 Binder。

#### 三. Linux下传统的进程间通信原理

了解 Linux IPC 相关的概念和原理有助于我们理解 Binder 通信原理。因此，在介绍 Binder 跨进程通信原理之前，我们先聊聊 Linux 系统下传统的进程间通信是如何实现。

##### 基本概念介绍

这里我们先从 Linux 中进程间通信涉及的一些基本概念开始介绍，然后逐步展开，向大家说明传统的进程间通信的原理。

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghuz8xxeeej30zk0n675v.jpg)

上图展示了 Liunx 中跨进程通信涉及到的一些基本概念：

- 进程隔离
- 进程空间划分：用户空间(User Space)/内核空间(Kernel Space)
- 系统调用：用户态/内核态

##### 进程隔离

简单的说就是操作系统中，进程与进程间内存是不共享的。两个进程就像两个平行的世界，A 进程没法直接访问 B 进程的数据，这就是进程隔离的通俗解释。A 进程和 B 进程之间要进行数据交互就得采用特殊的通信机制：进程间通信（IPC）。

##### 进程空间划分：用户空间(User Space)/内核空间(Kernel Space)

现在操作系统都是采用的虚拟存储器，对于 32 位系统而言，它的寻址空间（虚拟存储空间）就是 2 的 32 次方，也就是 4GB。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也可以访问底层硬件设备的权限。为了保护用户进程不能直接操作内核，保证内核的安全，操作系统从逻辑上将虚拟空间划分为用户空间（User Space）和内核空间（Kernel Space）。针对 Linux 操作系统而言，将最高的 1GB 字节供内核使用，称为内核空间；较低的 3GB 字节供各进程使用，称为用户空间。

**内核空间**：是系统内核运行的空间，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。**数据可共享，所有进程共用1个内核空间**。

**用户空间**：是用户程序运行的空间，没有接触物理内存或设备的权限。**数据不可共享**。

##### 系统调用：用户态与内核态

虽然从逻辑上进行了用户空间和内核空间的划分，但不可避免的用户空间需要访问内核资源，比如文件操作、访问网络等等。为了突破隔离限制，就需要借助**系统调用**来实现。**系统调用是用户空间访问内核空间的唯一方式**，保证了所有的资源访问都是在内核的控制下进行的，避免了用户程序对系统资源的越权访问，提升了系统安全性和稳定性。

Linux 使用两级保护机制：0 级供系统内核使用，3 级供用户程序使用。

当一个任务（进程）执行系统调用而陷入内核代码中执行时，称进程处于**内核运行态（内核态）**。此时处理器处于特权级最高的（0级）内核代码中执行。当进程处于内核态时，执行的内核代码会使用当前进程的内核栈。每个进程都有自己的内核栈。

当进程在执行用户自己的代码的时候，我们称其处于**用户运行态（用户态）**。此时处理器在特权级最低的（3级）用户代码中运行。

系统调用主要通过如下两个函数来实现：

```
copy_from_user() //将数据从用户空间拷贝到内核空间
copy_to_user() //将数据从内核空间拷贝到用户空间
```

##### 传统 IPC 通信原理

传统的 IPC 的做法是消息发送方将要发送的数据存放在内存缓存区中，通过系统调用进入内核态。然后内核程序在内核空间分配内存，开辟一块内核缓存区，调用 copy_from_user() 函数将数据从用户空间的内存缓存区拷贝到内核空间的内核缓存区中。同样的，接收方进程在接收数据时在自己的用户空间开辟一块内存缓存区，然后内核程序调用 copy_to_user() 函数将数据从内核缓存区拷贝到接收进程的内存缓存区。这样数据发送方进程和数据接收方进程就完成了一次数据传输，我们称完成了一次进程间通信。如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv0v787nwj30zk0ogn0j.jpg)

这种传统的 IPC 通信方式有两个问题：

1. 性能低下，一次数据传递需要经历：内存缓存区 --> 内核缓存区 --> 内存缓存区，需要 2 次数据拷贝；
2. 接收数据的缓存区由数据接收进程提供，但是接收进程并不知道需要多大的空间来存放将要传递过来的数据，因此只能开辟尽可能大的内存空间或者先调用 API 接收消息头来获取消息体的大小，这两种做法不是浪费空间就是浪费时间。

####  四、Binder跨进程通信原理

##### 动态内核可加载模块 && 内存映射

正如前面所说，跨进程通信是需要内核空间做支持的。传统的 IPC 机制如管道、Socket 都是内核的一部分，因此通过内核支持来实现进程间通信自然是没问题的。但是 Binder 并不是 Linux 系统内核的一部分，那怎么办呢？这就得益于 Linux 的**动态内核可加载模块**（Loadable Kernel Module，LKM）的机制；模块是具有独立功能的程序，它可以被单独编译，但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行。这样，Android 系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间通过这个内核模块作为桥梁来实现通信。

> 在 Android 系统中，这个运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块就叫 **Binder 驱动**（Binder Dirver）。

那么在 Android 系统中用户进程之间是如何通过这个内核模块（Binder 驱动）来实现通信的呢？这就不得不通道 Linux 下的另一个概念：**内存映射**。

Binder IPC 机制中涉及到的内存映射通过 **mmap()** 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是**将用户空间的一块内存区域映射到内核空间**。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。

**内存映射能减少数据拷贝次数**，实现用户空间和内核空间的高效互动。两个空间各自的修改能直接反映在映射的内存区域，从而被对方空间及时感知。也正因为如此，内存映射能够提供对进程间通信的支持。

##### Binder IPC 实现原理

Binder IPC 正是基于内存映射（mmap）来实现的，但是 mmap() 通常是用在有物理介质的文件系统上的。

比如进程中的用户区域是不能直接和物理设备打交道的，如果想要把磁盘上的数据读取到进程的用户区域，需要两次拷贝（磁盘-->内核空间-->用户空间）；通常在这种场景下 mmap() 就能发挥作用，通过在物理介质和用户空间之间建立映射，减少数据的拷贝次数，用内存读写取代I/O读写，提高文件读取效率。

而 Binder 并不存在物理介质，因此 Binder 驱动使用 mmap() 并不是为了在物理介质和用户空间之间建立映射，而是**用来在内核空间创建数据接收的缓存空间**。

一次完整的 Binder IPC 通信过程通常是这样：

1. 首先 Binder 驱动在内核空间创建一个**数据接收缓存区**；
2. 接着在内核空间开辟一块**内核缓存区**，建立**内核缓存区**和**内核中数据接收缓存区**之间的映射关系，以及**内核中数据接收缓存区**和**接收进程用户空间地址**的映射关系；
3. 发送方进程通过系统调用 copy_from_user() 将数据 copy 到内核中的**内核缓存区**，由于**内核缓存区和接收进程的用户空间都映射到数据接收缓存区，所以两者也存在内存映射关系**，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。

如下图：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv2qmtimuj30zk0pdwis.jpg)

**总结**：

1. Linux的虚拟内存机制导致内存的隔离，进而导致进程隔离
2. 进程隔离的出现导致对内存的操作被划分为用户空间和内核空间
3. 用户空间需要跨权限去访问内核空间，必须使用系统调用去实现
4. 系统调用需要借助内核模块/驱动去完成

前三步决定了进程间通讯需要借助**内核模块/驱动**去实现，而 Binder 驱动就是内核模块/驱动中用来实现进程间通讯的。

#### 五、Binder 通信模型

介绍完 Binder IPC 的底层通信原理，接下来我们看看实现层面是如何设计的。

一次完整的进程间通信必然至少包含两个进程，通常我们称通信的双方分别为客户端进程（Client）和服务端进程（Server），由于进程隔离机制的存在，通信双方必然需要借助 Binder 来实现。

##### Client/Server/ServiceManager/驱动

前面我们介绍过，Binder 是基于 C/S 架构的。由一系列的组件组成，包括Client、Server、ServiceManager、Binder 驱动。其中 Client、Server、Service Manager 运行在用户空间，Binder 驱动运行在内核空间。其中 Service Manager 和 Binder 驱动由系统提供，而 Client、Server 由应用程序来实现。Client、Server 和 ServiceManager 均是通过系统调用 open、mmap 和 ioctl 来访问设备文件 /dev/binder，从而实现与 Binder 驱动的交互来间接的实现跨进程通信。

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv3pv6tj0j30zk0kv77j.jpg)

Client、Server、ServiceManager、Binder 驱动这几个组件在通信过程中扮演的角色就如同互联网中服务器（Server）、客户端（Client）、DNS域名服务器（ServiceManager）以及路由器（Binder 驱动）之前的关系。

[Android Binder 设计与实现](http://blog.csdn.net/universus/article/details/6211589)一文中对 Client、Server、ServiceManager、Binder 驱动有很详细的描述，以下是部分摘录：

> **Binder 驱动**
> Binder 驱动就如同路由器一样，是整个通信的核心；驱动负责进程之间 Binder 通信的建立，Binder 在进程之间的传递，Binder 引用计数管理，数据包在进程之间的传递和交互等一系列底层支持。 
>
> **ServiceManager 与实名 Binder**
> ServiceManager 和 DNS 类似，作用是将字符形式的 Binder 名字转化成 Client 中对该 Binder 的引用，使得 Client 能够通过 Binder 的名字获得对 Binder 实体的引用。注册了名字的 Binder 叫实名 Binder，就像网站一样除了除了有 IP 地址意外还有自己的网址。Server 创建了 Binder，并为它起一个字符形式，可读易记得名字，将这个 Binder 实体连同名字一起以数据包的形式通过 Binder 驱动发送给 ServiceManager ，通知 ServiceManager 注册一个名为“张三”的 Binder，它位于某个 Server 中。驱动为这个穿越进程边界的 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager。ServiceManger 收到数据后从中取出名字和引用填入查找表。
>
> 细心的读者可能会发现，ServierManager 是一个进程，Server 是另一个进程，Server 向 ServiceManager 中注册 Binder 必然涉及到进程间通信。当前实现进程间通信又要用到进程间通信，这就好像蛋可以孵出鸡的前提却是要先找只鸡下蛋！Binder 的实现比较巧妙，就是预先创造一只鸡来下蛋。ServiceManager 和其他进程同样采用 Bidner 通信，ServiceManager 是 Server 端，有自己的 Binder 实体，其他进程都是 Client，需要通过这个 Binder 的引用来实现 Binder 的注册，查询和获取。ServiceManager 提供的 Binder 比较特殊，它没有名字也不需要注册。当一个进程使用 BINDER_SET_CONTEXT_MGR 命令将自己注册成 ServiceManager 时 Binder 驱动会自动为它创建 Binder 实体（**这就是那只预先造好的那只鸡**）。其次这个 Binder 实体的引用在所有 Client 中都固定为 0 而无需通过其它手段获得。也就是说，一个 Server 想要向 ServiceManager 注册自己的 Binder 就必须通过这个 0 号引用和 ServiceManager 的 Binder 通信。类比互联网，0 号引用就好比是域名服务器的地址，你必须预先动态或者手工配置好。要注意的是，这里说的 Client 是相对于 ServiceManager 而言的，一个进程或者应用程序可能是提供服务的 Server，但对于 ServiceManager 来说它仍然是个 Client。
>
> **Client 获得实名 Binder 的引用**
> Server 向 ServiceManager 中注册了 Binder 以后， Client 就能通过名字获得 Binder 的引用了。Client 也利用保留的 0 号引用向 ServiceManager 请求访问某个 Binder: 我申请访问名字叫张三的 Binder 引用。ServiceManager 收到这个请求后从请求数据包中取出 Binder 名称，在查找表里找到对应的条目，取出对应的 Binder 引用作为回复发送给发起请求的 Client。从面向对象的角度看，Server 中的 Binder 实体现在有两个引用：一个位于 ServiceManager 中，一个位于发起请求的 Client 中。如果接下来有更多的 Client 请求该 Binder，系统中就会有更多的引用指向该 Binder ，就像 Java 中一个对象有多个引用一样。

- **Client**：客户端。
- **Server**：服务端。
- **ServiceManager**（如同DNS域名服务器）：服务的管理者，提供服务的注册和查询。将 Binder 名字转换为 Client 中对该 Binder 的引用，使得 Client 可以通过 Binder 名字获得 Server 中 Binder 实体的引用，本身也是一个 Binder 服务。
- **Binder驱动**（如同路由器）：进程通信的介质，负责进程之间 binder 通信的建立，传递，计数管理以及数据的传递交互等底层支持。

##### Binder通信过程

至此，我们大致能总结出 Binder 通信过程：

1. 首先，一个进程使用 BINDER_SET_CONTEXT_MGR 命令通过 Binder 驱动将自己注册成为 ServiceManager；
2. Server 通过驱动向 ServiceManager 中注册 Binder（Server 中的 Binder 实体），表明可以对外提供服务。驱动为这个 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager，ServiceManger 将其填入查找表。
3. Client 通过名字，在 Binder 驱动的帮助下从 ServiceManager 中获取到对 Binder 实体的引用，通过这个引用就能实现和 Server 进程的通信。

架构图如下所示：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghv4ottvlkj30pl0cgjuv.jpg)

可以看出无论是注册服务和获取服务的过程都需要ServiceManager，需要注意的是此处的Service Manager是指Native层的ServiceManager（C++），并非指framework层的ServiceManager(Java)。ServiceManager是整个Binder通信机制的大管家，是Android进程间通信机制Binder的守护进程，要掌握Binder机制，首先需要了解系统是如何首次[启动Service Manager](http://gityuan.com/2015/11/07/binder-start-sm/)。当Service Manager启动之后，Client端和Server端通信时都需要先[获取Service Manager](http://gityuan.com/2015/11/08/binder-get-sm/)接口，才能开始通信服务。

图中Client/Server/ServiceManage之间的相互通信都是基于Binder机制。既然基于Binder机制通信，那么同样也是C/S架构，则图中的3大步骤都有相应的Client端与Server端。

1. **[注册服务(addService)](http://gityuan.com/2015/11/14/binder-add-service/)**：Server进程要先注册Service到ServiceManager。该过程：Server是客户端，ServiceManager是服务端。
2. **[获取服务(getService)](http://gityuan.com/2015/11/15/binder-get-service/)**：Client进程使用某个Service前，须先向ServiceManager中获取相应的Service。该过程：Client是客户端，ServiceManager是服务端。
3. **使用服务**：Client根据得到的Service信息建立与Service所在的Server进程通信的通路，然后就可以直接与Service交互。该过程：client是客户端，server是服务端。

图中的Client,Server,Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与[Binder驱动](http://gityuan.com/2015/11/01/binder-driver/)进行交互的，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client,Server,Service Manager位于用户空间。Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Server是Android的应用层，开发人员只需自定义实现client、Server端，借助Android的基本平台架构便可以直接进行IPC通信。

注意：

- 每个 Binder 的 Server 进程会创建很多线程来处理 Binder 请求
- Binder 模型的线程管理 采用 Binder 驱动的线程池，并由 Binder 驱动自身进行管理，而不是由 Server 进程来管理的
- 一个进程的 Binder 线程数默认最大是16，超过的请求会被阻塞等待空闲的 Binder 线程。如使用ContentProvider 时，它的 CRUD 方法只能同时有16个线程同时工作。

##### C/S模式

BpBinder(客户端)和 BBinder(服务端)都是 Android 中 Binder 通信相关的代表，它们都从 IBinder 类中派生而来，它们属于 Native 层，关系图如下：

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghw4y8di3ij30a107qq37.jpg)

- client端：BpBinder.transact()来发送事务请求，是 Binder 引用对象；
- server端：BBinder.onTransact()会接收到相应事务，是 Binder 实体对象。

##### Binder通信中的代理模式

我们已经解释清楚 Client、Server 借助 Binder 驱动完成跨进程通信的实现机制了，但是还有个问题会让我们困惑。A 进程想要 B 进程中某个对象（object）是如何实现的呢？毕竟它们分属不同的进程，A 进程 没法直接使用 B 进程中的 object。

前面我们介绍过跨进程通信的过程都有 Binder 驱动的参与，因此在数据流经 Binder 驱动的时候驱动会对数据做一层转换。当 A 进程想要获取 B 进程中的 object 时，驱动并不会真的把 object 返回给 A，而是返回了一个跟 object 看起来一模一样的代理对象 objectProxy，这个 objectProxy 具有和 object 一摸一样的方法，但是这些方法并没有 B 进程中 object 对象那些方法的能力，这些方法只需要把请求参数交给驱动即可。对于 A 进程来说和直接调用 object 中的方法是一样的。

当 Binder 驱动接收到 A 进程的消息后，发现这是个 objectProxy 就去查询自己维护的表单，一查发现这是 B 进程 object 的代理对象。于是就会去通知 B 进程调用 object 的方法，并要求 B 进程把返回结果发给自己。当驱动拿到 B 进程的返回结果后就会转发给 A 进程，一次通信就完成了。

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghw5dr45igj30zk0iwjtm.jpg)

- Binder对象：Binder机制中进行进程间通讯的对象，对于Server端为Binder本地对象，对于Client端为Binder代理对象。
- Binder驱动：Binder机制中进行进程间通讯的介质，Binder驱动会对具有跨进程传递能力的对象做特殊处理，自动完成代理对象和本地对象的转换。因此在驱动中保存了每一个跨越进程的Binder对象的相关信息，Binder本地对象（或Binder实体）保存在binder_node的数据结构，Binder代理对象（或Binder引用/句柄）保存在binder_ref的数据结构。

##### Binder的完整定义

现在我们可以对 Binder 做个更加全面的定义了：

- 从进程间通信的角度看，Binder 是一种进程间通信的机制；
- 从 Server 进程的角度看，Binder 指的是 Server 中的 Binder 实体对象；
- 从 Client 进程的角度看，Binder 指的是对 Binder 代理对象，是 Binder 实体对象的一个远程代理；
- 从传输过程的角度看，Binder 是一个可以跨进程传输的对象；Binder 驱动会对这个跨越进程边界的对象做特殊处理，自动完成代理对象和本地对象之间的转换；
- 从 Android Driver 层的角度看，Binder 是一种虚拟的物理设备，它的设备驱动是 /dev/binder；
- 从 Android Framework 来讲，Binder是 Service Manager 连接各种 Manager 和对应的 ManagerService 的桥梁。

#### 六、AIDL

通常我们在做开发时，实现进程间通信用的最多的就是 AIDL——Android接口定义语言。当我们定义好 AIDL 文件，在编译时编译器会帮我们生成代码实现 IPC 通信。借助 AIDL 编译以后的代码能帮助我们进一步理解 Binder IPC 的通信原理。基于Binder，Android还实现了其他的IPC方式，比如 Messenger 和 ContentProvider 。

但是无论是从可读性还是可理解性上来看，编译器生成的代码对开发者并不友好。比如一个 BookManager.aidl 文件对应会生成一个 BookManager.java 文件，这个 java 文件包含了一个 BookManager 接口、一个 Stub 静态的抽象类和一个 Proxy 静态类。Proxy 是 Stub 的静态内部类，Stub 又是 BookManager 的静态内部类，这就造成了可读性和可理解性的问题。

> Android 之所以这样设计其实是有道理的，因为当有多个 AIDL 文件的时候把 BookManager、Stub、Proxy 放在同一个文件里能有效避免 Stub 和 Proxy 重名的问题。

##### 各Java类职责描述

在正式编码实现跨进程调用之前，先介绍下实现过程中用到的一些类。了解了这些类的职责，有助于我们更好的理解和实现跨进程通信。

- **IBinder**：是一个接口，定义了Java层Binder通信的一些规则；提供了transact方法来调用远程服务，代表了一种跨进程通信的能力。只要实现了这个接口，这个对象就能跨进程传输。这是 Binder 驱动底层支持的，Binder 驱动会自动完成不同进程 Binder 本地对象以及Binder 代理对象的转换。
- **IInterface**：Client 端与 Server 端的调用契约，代表了 Server 进程对象具备什么样的能力（能提供哪些方法，其实对应的就是 AIDL 文件中定义的接口方法），所有的服务提供者，必须继承这个接口。
- **Binder**：Java 层的 Binder 类，代表的就是 Binder 本地对象。BinderProxy 类是代表远程进程的 Binder 本地对象的代理；这两个类都继承自 IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder 驱动会自动完成这两个对象的转换。
- **Stub**：AIDL 的时候，编译工具会给我们生成一个名为 Stub 的静态内部类；这个类继承了 Binder，说明它是一个 Binder 本地对象，它实现了 IInterface 接口，表明它具有 Server 承诺给 Client 的能力；Stub 是一个抽象类，具体的 IInterface 的相关实现需要开发者自己实现。这里使用了策略模式。
- **Proxy**：服务端 Binder 本地对象代理，客户端通过这个类调用服务端的方法。
- **Parcel**：是一个容器，它主要用于存储序列化数据，然后可以通过 Binder 在进程间传递这些数据。

##### AIDL语法

- 文件类型：用AIDL书写的文件的后缀是 .aidl，而不是 .java。

- 数据类型：AIDL默认支持一些数据类型，在使用这些数据类型的时候是不需要导包的，但是除了这些类型之外的数据类型，在使用之前必须导包，**就算目标文件与当前正在编写的 .aidl 文件在同一个包下**——在 Java 中，这种情况是不需要导包的。比如，现在我们编写了两个文件，一个叫做 **Book.java** ，另一个叫做 **BookManager.aidl**，它们都在 **com.prsuit.androidlearnsample** 包下 ，现在我们需要在 .aidl 文件里使用 Book 对象，那么我们就必须在 .aidl 文件里面写上 **`import com.prsuit.androidlearnsample.Book;`** 哪怕 .java 文件和 .aidl 文件就在一个包下。 
  默认支持的数据类型包括： 
  - Java 中的基本数据类型( byte，short(不支持short，编译不通过)，int，long，float，double，boolean，char)，String 和 CharSequence类型。
  - List 和 Map：元素必须是 AIDL 支持的数据类型之一，Server 端具体的类里必须是 ArrayList 或者 HashMap。
  - AILD：其他 AIDL 生成的接口。
  - 除了默认支持的数据类型外，AIDL 还支持自定义实现 Parcelable 接口的数据类型。

- 定向tag：方法内如果有传输载体，就必须指明定向 tag ([in,out,inout](http://blog.csdn.net/luoyanglizi/article/details/51958091))。
  - in：**客户端数据对象流向服务端**，并且服务端对该数据对象的修改不会影响到客户端。
  - out： **数据对象由服务端流向客户端**，（客户端传递的数据对象此时服务端收到的对象内容为空，服务端可以对该数据对象修改，并传给客户端）。
  - inout：**数据可在服务端与客户端之间双向流通**，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务端对该对象的任何变动。（但是不建议用此 tag，会增加开销）。

- 两种AIDL文件：**一类是用来定义 parcelable 对象**，以供其他 AIDL 文件使用非默认支持的数据类型。**一类是用来定义方法接口**，以供系统使用来完成跨进程通信的。**所有的非默认支持数据类型必须通过第一类AIDL文件定义才能被使用。**

##### 使用步骤

**创建实体类**

实体类要实现 Parcelable 接口，且要手动添加 `readFromParcel` 方法（不然只支持 in 的定向tag）。

注：**若AIDL文件中涉及到的所有数据类型均为默认支持的数据类型，则无此步骤。因为默认支持的那些数据类型都是可序列化的。**

```java
public class Book implements Parcelable {
    private String name;
    private int price;

    public Book(){
    }

    public Book(Parcel in) {
        this.name = in.readString();
        this.price = in.readInt();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(price);
    }

    // 该方法不是Parcelable自动生成的，需要自己手动添加，
    // 如果不添加，则在使用AIDL时只支持 in 的定向tag
    // 如果添加了，则支持 out、inout
    public void readFromParcel(Parcel dest){
      //注意，此处的读值顺序应当是和writeToParcel()方法中一致的
        name = dest.readString();
        price = dest.readInt();
    }
      ...
      //get set方法
      ...
}
```

**创建定义parcelable对象的AIDL文件**

因为AIDL这个语言的规范就是 aidl 文件，所以我们必须将`Book`实体类转为 aidl 文件，供其它 aidl 的调用与交互。**Book.aidl与Book.java的包名要是一样的**。

```java
// Book.aidl
//第一类AIDL文件
//这个文件的作用是引入了一个序列化对象 Book 供其他的AIDL文件使用
//注意：Book.aidl与Book.java的包名应当是一样的

package com.prsuit.androidlearnsample;

// Declare any non-default types here with import statements
//注意parcelable是小写
parcelable Book;
```

如何在保证两个文件包名一样的情况下，让系统能够找到我们的 java 文件？思路：要么让系统来 aidl 包里面来找 java 文件，要么把 java 文件放到系统能找到的地方去，也即放到 java 包里面去。这两种方式具体做法：

- 修改 build.gradle 文件，通过 sourceSets 配置把 java 的访问路径设置成了 java 包和 aidl 包，在 android{} 中加上下面的内容：

  ```java
  sourceSets {
      main {
          java.srcDirs = ['src/main/java', 'src/main/aidl']
      }
  }
  ```

- 把 java 文件放到 java 包下去：把 Book.java 放到 java 包里任意一个包下，保持其包名不变，与 Book.aidl 一致。只要它的包名不变，Book.aidl 就能找到 Book.java ，而只要 Book.java 在 java 包下，那么系统也是能找到它的。注意在移植相关 .aidl 文件时，不能直接把整个 aidl 文件夹拿过去完事了，还要单独将 .java 文件放到 java 文件夹里去。

**创建定义方法接口的AIDL文件**

该文件是 Client 端与 Server 端的调用契约，定义 Server 进程对象具备什么样的能力。

注：**接口方法 aidl 中不能存在同方法名不同参数的方法。**

```java
// IBookManager.aidl
package com.prsuit.androidlearnsample;
import com.prsuit.androidlearnsample.Book;
// Declare any non-default types here with import statements

interface IBookManager {
  
  //有返回值前都不需要加任何东西，不管是什么数据类型
  List<Book> getBooks();

  //Java基本类型以及String，CharSequence的 tag 默认且只能是 in
  //传参时除了Java基本类型以及String，CharSequence之外的类型
  //如果有传输对象载体，就必须指明定向tag(in,out,inout)
  Book addBookIn(in Book book);//客户端->服务端
  //out和inout都需要重写MessageBean的readFromParcel方法
  Book addBookOut(out Book book);//服务端->客户端
  Book addBookInOut(inout Book book);//客户端<->服务端
}
```

**编写服务端代码**

在我们写完AIDL文件并 Make Project 项目之后，编译器会根据AIDL文件为我们生成一个与AIDL文件同名的 .java 文件，这个 .java 文件才是与我们的跨进程通信密切相关的东西。在服务端实现AIDL中定义的方法接口的具体逻辑，创建一个服务，用来处理客户端发来的请求。

```java
/**
 * @Description: 服务端的AIDLService.java
 */
public class AIDLService extends Service {
    private final String TAG = "AIDLService";
    private List<Book> mBooks = new ArrayList<>();

    //由AIDL文件生成的IBookManager
    private IBookManager.Stub mBookManager = new IBookManager.Stub() {
        @Override
        public List<Book> getBooks() throws RemoteException {
          synchronized(this) {
            Log.e(TAG, "getBooks: "+ mBooks.toString());
            if (mBooks != null){
                return mBooks;
            }
            return new ArrayList<>();
          }
        }

        @Override
        public Book addBookIn(Book book) throws RemoteException {
          synchronized(this) {
            Log.e(TAG, "addBookIn: 接收到的对象-->"+book.toString() );
            if (book == null){
                Log.e(TAG, "addBookIn: book is null in In");
                book = new Book();
            }
            //尝试修改book的参数，主要是为了观察其到客户端的反馈
            book.setPrice(2333);
            if (!mBooks.contains(book)){
                mBooks.add(book);
            }
            Log.e(TAG, "addBookIn: now the list is "+mBooks.toString() );
            return book;
          }
        }

        @Override
        public Book addBookOut(Book book) throws RemoteException {
          synchronized(this) {
            Log.e(TAG, "addBookOut: 接收到的对象-->"+book.toString() );
            if (book == null){
                Log.e(TAG, "addBookOut: book is null in Out");
                book = new Book();
            }
            //尝试修改book的参数，主要是为了观察其到客户端的反馈
            book.setPrice(2333);
            if (!mBooks.contains(book)){
                mBooks.add(book);
            }
            Log.e(TAG, "addBookOut: now the list is "+ mBooks.toString());
            return book;
          }
        }

        @Override
        public Book addBookInOut(Book book) throws RemoteException {
          synchronized(this){
            Log.e(TAG, "addBookInOut: 接收到的对象-->"+book.toString() );
            if (book == null){
                Log.e(TAG, "addBookInOut: book is null in InOut");
                book = new Book();
            }
            //尝试修改book的参数，主要是为了观察其到客户端的反馈
            book.setPrice(2333);
            if (!mBooks.contains(book)){
                mBooks.add(book);
            }
            Log.e(TAG, "addBookInOut: now the list is "+ mBooks.toString());
            return book;
          }
        }
    };

    @Override
    public void onCreate() {
        Log.e(TAG, "onCreate: AIDLService" );
        super.onCreate();
        Book book = new Book();
        book.setName("Android 开发艺术探索");
        book.setPrice(20);
        mBooks.add(book);
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.e(TAG, "onBind: AIDLService" );
        return mBookManager;
    }
}
```

整体的代码结构很清晰，大致可以分为三块：第一块是**初始化**。在 onCreate() 方法里面我进行了一些数据的初始化操作。第二块是**重写 IBookManager.Stub 中的方法**。在这里面提供AIDL里面定义的方法接口的具体实现逻辑。第三块是**重写 onBind() 方法**，返回 IBookManager.Stub 对象。

接下来在 Manefest 文件里面注册这个我们写好的 Service ，添加 action 以便客户端可以隐式启动该服务。如果在同一个项目中可通过设置`android:process` 处于不同进程。

```java
<service
    android:name=".service.AIDLService"
    android:process=":remote"
    android:exported="true">
        <intent-filter>
            <action android:name="com.prsuit.aidl.bookmanager"/>
            <category android:name="android.intent.category.DEFAULT"/>
        </intent-filter>
</service>
```

**编写客户端代码**

如果是在不同项目中，则需要拷贝 AIDL 文件夹和相关 Java 文件，aidl 移植完后，客户端主要工作是连接上服务端，调用服务端的方法。

```java
/**
 * 客户端的AIDLActivity.java
 */
public class AIDLActivity extends AppCompatActivity {
    private final String TAG = "AIDLActivity";

    private IBookManager mBookManager = null;//服务端binder对象
    private boolean mBound = false;//是否连接上服务端
    private List<Book> mBooks;
    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBookManager = IBookManager.Stub.asInterface(service);
            mBound = true;
            if (mBookManager != null){
                try {
                  mBooks = mBookManager.getBooks();
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mBound = false;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_a_i_d_l);
    }

    /**
     * 调用服务端的addBookIn方法
     * @param view
     */
    public void addBookIn(View view){
        if (!mBound){
            toBindService();
            return;
        }
        if (mBookManager == null){
            return;
        }
        Book book = new Book();
        book.setName("App 开发In");
        book.setPrice(30);
        try {
            //获得服务端执行方法的返回值，并打印输出
            Book returnBook = mBookManager.addBookIn(book);
            Log.e(TAG, "addBookIn: Client book："+book.toString() );
            Log.e(TAG, "addBookIn: server 返回book："+returnBook.toString() );
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    /**
     * 调用服务端的addBookOut方法
     * @param view
     */
    public void addBookOut(View view){
        if (!mBound){
            toBindService();
            return;
        }
        if (mBookManager == null){
            return;
        }
        Book book = new Book();
        book.setName("App 开发Out");
        book.setPrice(30);
        try {
            //获得服务端执行方法的返回值，并打印输出
            Book returnBook = mBookManager.addBookOut(book);
            Log.e(TAG, "addBookOut: Client book："+book.toString() );
            Log.e(TAG, "addBookOut: server 返回book："+returnBook.toString() );
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    /**
     * 调用服务端的addBookInOut方法
     * @param view
     */
    public void addBookInOut(View view){
        if (!mBound){
            toBindService();
            return;
        }
        if (mBookManager == null){
            return;
        }
        Book book = new Book();
        book.setName("App 开发InOut");
        book.setPrice(30);
        try {
            //获得服务端执行方法的返回值，并打印输出
            Book returnBook = mBookManager.addBookInOut(book);
            Log.e(TAG, "addBookInOut: Client book："+book.toString() );
            Log.e(TAG, "addBookInOut: server 返回book："+returnBook.toString() );
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onStart() {
        super.onStart();
        if (!mBound){
            toBindService();
        }
    }

    /**
     * 尝试与服务端建立连接
     */
    private void toBindService() {
        Intent intent = new Intent();
        intent.setAction("com.prsuit.aidl.bookmanager");
        //5.0版本后隐式启动服务报错，必须给Intent设置包名
        intent.setPackage("com.prsuit.androidlearnsample");
        bindService(intent,mServiceConnection,BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (mBound){
            unbindService(mServiceConnection);
            mBound = false;
        }
    }
}
```

同样很清晰，首先建立连接，然后在 ServiceConnection 里面获取 IBookManager 对象，接着通过它来调用服务端的方法。

##### 原理分析

先用一张图整体描述这个AIDL从客户端(Client)发起请求至服务端(Server)相应的工作流程，我们可以看出整体的核心就是Binder。

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghxfotybjhj30zs0dy75l.jpg)

首先看下客户端的 IBookManager 对象是怎么来的：

```java
 public void onServiceConnected(ComponentName name, IBinder service) {
            mBookManager = IBookManager.Stub.asInterface(service);
```

**asInterface()**：

> 用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的【如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象】

注意到方法的传参：IBinder service ，这是个 BinderProxy 对象，接下来顺藤摸瓜去看下这个 BookManager.Stub.asInterface() 是怎么回事：

```java
 public static com.prsuit.androidlearnsample.IBookManager asInterface(android.os.IBinder obj)
    {
     //验空
      if ((obj==null)) {
        return null;
      }
    //DESCRIPTOR = "com.prsuit.androidlearnsample.IBookManager"，搜索本地是否有可用的对象
    //如果找到了就说明 Client 和 Server 在同一进程，那么这个 binder 本身就是 Binder 本地对象
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.prsuit.androidlearnsample.IBookManager))) {
        return ((com.prsuit.androidlearnsample.IBookManager)iin);
      }
    //如果本地没有的话，说明是 binder 是个远程对象，就新建一个代理对象Proxy返回
      return new com.prsuit.androidlearnsample.IBookManager.Stub.Proxy(obj);
    }
```

可以看出通过`DESCRIPTOR`标识去查找本地是否有可用对象，如果有则是同一进程，那么就返回Stub对象本身，如果没找到则是不同进程，返回Stub的代理内部类Proxy。

**Proxy**：

看看Stub的代理内部类IBookManager.Stub.Proxy 类，客户端最终是通过这个类与服务端进行通信。

```java
 private static class Proxy implements com.prsuit.androidlearnsample.IBookManager
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
         //此处的 remote 正是前面我们提到的 IBinder service
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
        return DESCRIPTOR;//Binder的唯一标识
      }
     
      @Override 
   public java.util.List<com.prsuit.androidlearnsample.Book> getBooks() throws android.os.RemoteException
      {
         //_data用来存储流向服务端的数据流
        android.os.Parcel _data = android.os.Parcel.obtain();
        //_reply用来存储服务端流回客户端的数据流
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.util.List<com.prsuit.androidlearnsample.Book> _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          //调用 transact() 方法将方法id和两个 Parcel 容器传过去
          boolean _status = mRemote.transact(Stub.TRANSACTION_getBooks, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getBooks();
          }
          _reply.readException();
           //从_reply中取出服务端执行方法的结果
          _result = _reply.createTypedArrayList(com.prsuit.androidlearnsample.Book.CREATOR);
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        //将结果返回
        return _result;
      }
      @Override 
   public com.prsuit.androidlearnsample.Book addBookIn(com.prsuit.androidlearnsample.Book book) throws android.os.RemoteException
      {
       //省略
      }
   ...
}
```

 **transact()** ：

> `transact`方法运行在客户端，首先它创建该方法所需要的输入型Parcel对象_data、输出型Parcel对象_reply;
> 接着调用绑定服务传来的IBinder对象的transact方法来发起远程过程调用（RPC）请求，同时当前线程挂起;
> 然后服务端的`onTransact`方法会被调用，直到RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果，也就是返回_reply中的数据。

这是客户端和服务端通信的核心方法。调用这个方法之后，客户端将会挂起当前线程，等候服务端执行完相关任务后通知并接收返回的 _reply 数据流。关于这个方法的传参，这里需要说明的地方：

- 方法 ID ：transact() 方法的第一个参数是一个方法 ID ，这个是客户端与服务端约定好的给方法的编码，彼此一一对应。在AIDL文件转化为 .java 文件的时候，系统将会自动给AIDL文件里面的每一个方法自动分配一个方法 ID。

- 关于 _data 与 _reply 对象：一般来说，我们会将方法的传参的数据存入_data 中，而将方法的返回值的数据存入 _reply 中——在没涉及定向 tag 的情况下。

- 第四个参数：transact() 方法的第四个参数是一个 int 值，它的作用是设置进行 IPC 的模式，为 0 表示数据可以双向流通，即 _reply 流可以正常的携带数据回来，如果为 1 的话那么数据将只能单向流通，从服务端回来的 _reply 流将不携带任何数据。 

- 关于 Parcel ：简单的来说，Parcel 是一个用来存放和读取数据的容器。我们可以用它来进行客户端和服务端之间的数据传输，当然，它能传输的只能是可序列化的数据。

  注：AIDL生成的 .java 文件的这个参数均为 0。

客户端的一般工作流程：

1. 生成 _data 和 _reply 数据流，并向 _data 中存入客户端的数据。
2. 通过 transact() 方法将它们传递给服务端，并请求服务端调用指定方法。
3. 接收 _reply 数据流，并从中取出服务端传回来的数据。

**Stub**：

接下来看服务端Stub类，在服务端有一个方法` onTransact()`来接收客户端传过来的参数并进行处理。这个`onTransact`方法就是服务端处理的核心，接收到客户端的请求，并且通过客户端携带的参数，执行完服务端的方法，返回结果。IBookManager.Stub类：

```java
public static abstract class Stub extends android.os.Binder implements com.prsuit.androidlearnsample.IBookManager
  {
    private static final java.lang.String DESCRIPTOR = "com.prsuit.androidlearnsample.IBookManager";
    /** Construct the stub at attach it to the interface. */
    public Stub()
    {
      this.attachInterface(this, DESCRIPTOR);
    } 
  @Override 
  public android.os.IBinder asBinder()
    {
      return this;
    }
  @Override 
  public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
    {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code)
      {
        case INTERFACE_TRANSACTION:
        {
          reply.writeString(descriptor);
          return true;
        }
        case TRANSACTION_getBooks:
        {
          data.enforceInterface(descriptor);
          //调用 this.getBooks() 方法，在这里开始执行具体的事务逻辑
          //result 列表为调用 getBooks() 方法的返回值
          java.util.List<com.prsuit.androidlearnsample.Book> _result = this.getBooks();
          reply.writeNoException();
          //将方法执行的结果写入 reply 
          reply.writeTypedList(_result);
          return true;
        }
        case TRANSACTION_addBookIn:
        {
          data.enforceInterface(descriptor);
          com.prsuit.androidlearnsample.Book _arg0;
          if ((0!=data.readInt())) {
            _arg0 = com.prsuit.androidlearnsample.Book.CREATOR.createFromParcel(data);
          }
          else {
            _arg0 = null;
          }
          com.prsuit.androidlearnsample.Book _result = this.addBookIn(_arg0);
          reply.writeNoException();
          if ((_result!=null)) {
            reply.writeInt(1);
            _result.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
          }
          else {
            reply.writeInt(0);
          }
          return true;
        }
          ...
        default:
        {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }
}
```

**onTransact()**：

> `onTransact`方法运行在服务端中的Binder线程池中
> 客户端发起跨进程请求时，远程请求会通过系统底层封装后交给此方法来处理。
> 如果此方法返回false,那么客户端的请求就会失败。

在 onTransact 方法中直接调用服务端的具体方法实现，如果目标方法有参数的话，就从data取出目标方法所需的参数，当目标方法执行完毕后，如果目标方法有返回值，就向reply中写入返回值。

服务端的一般工作流程：

1. 获取客户端传过来的数据，根据方法 ID 执行相应操作。
2. 将传过来的数据取出来，调用本地写好的对应方法。
3. 将需要回传的数据写入 reply 流，传回客户端。

**Stub与Proxy区别**

Proxy与Stub不一样，虽然他们都既是Binder又是IInterface，不同的是Stub采用的是继承（is 关系），Proxy采用的是组合（has 关系）。他们均实现了所有的IInterface函数，不同的是，Stub又使用策略模式调用的是虚函数（待子类实现），而Proxy则使用组合模式。为什么Stub采用继承而Proxy采用组合？事实上，Stub本身is一个IBinder（Binder），它本身就是一个能跨越进程边界传输的对象，所以它得继承IBinder实现transact这个函数从而得到跨越进程的能力（这个能力由驱动赋予）。Proxy类使用组合，是因为他不关心自己是什么，它也不需要跨越进程传输，它只需要拥有这个能力即可，要拥有这个能力，只需要保留一个对IBinder的引用。在Stub类里面，asBinder返回this，在Proxy里面返回的是持有的组合类IBinder的引用。

一个需要跨进程传递的对象一定继承自 IBinder，如果是 Binder 本地对象，那么一定继承 Binder 实现IInterface，如果是代理对象，那么就实现了 IInterface 并持有了 IBinder 引用。

**Binder传输数据的大小限制**

普通的由Zygote孵化而来的用户进程，所映射的**Binder内存大小是不到1M的**，准确说是 1x1024x1024) - (4096 x2)，这个限制定义在ProcessState类中，如果传输说句超过这个大小，系统就会报错，而在内核中，其实也有个限制，是4M，不过由于APP中已经限制了不到1M。

有个特殊的进程ServiceManager进程，它为自己申请的Binder内核空间是128K，这个同ServiceManager的用途是分不开的，ServcieManager主要面向系统Service，只是简单的提供一些addServcie，getService的功能，不涉及多大的数据传输，因此不需要申请多大的内存。

**ServiceManager addService的限制**

ServiceManager 其实主要的面向对象是**系统服务**，比如AMS、WMS、PKMS服务等，并非所有服务都能通过addService 添加到 ServiceManager，在通过ServiceManager添加服务的时候，是有些权限校验的，大部分系统服务都是由 SystemServer 进程中添加到ServiceManager中去的。而APP通过的 bindService 启动的 Binder 服务其实是**由 SystemServer 的 ActivityManagerService **负责管理。

### 总结

Binder通信的实质是利用内存映射，将用户进程的内存地址和内核的内存地址映射为同一块物理地址，也就是说他们使用的同一块物理空间。 

![](https://tva1.sinaimg.cn/large/007S8ZIlly1ghyk9gwafcj30qo0t5qc1.jpg)

[Demo](https://github.com/prsuit/Android-Learn-Sample)

引用文章：

> [写给 Android 应用工程师的 Binder 原理剖析](https://juejin.im/post/6844903589635162126)
>
> [图文详解 Android Binder跨进程通信的原理](https://www.jianshu.com/p/4ee3fd07da14)
>
> [Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)
>
> [Android Bander设计与实现 - 设计篇](https://blog.csdn.net/universus/article/details/6211589)
>
> [Android跨进程通信IPC之13——Binder总结](https://www.jianshu.com/p/485233919c15)
>
> [Android 深入浅出AIDL（一）](https://blog.csdn.net/qian520ao/article/details/78072250)
>
> [Android 深入浅出AIDL（二）](https://blog.csdn.net/qian520ao/article/details/78074983)
>
> [你真的理解AIDL中的in，out，inout么？](https://blog.csdn.net/luoyanglizi/article/details/51958091)

