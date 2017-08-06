# Binder

Binder 实现机制

Stub 类中 asInterface 函数作用

BBinder 和 BpBinder 的具体含义和区别

从 java 到 framework 再到 kenral 层



## 1. Binder概述

1. 从 IPC 角度来说：Binder 是 Android 中的一种跨进程通信方式，该通信方式在 linux 中没有，是 Android 独有；
2. 从 Android Driver 层：Binder 还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder；
3. 从 Android Native 层：Binder 是创建 Service Manager 以及 BpBinder/BBinder 模型，搭建与 binder 驱动的桥梁；
4. 从 Android Framework 层：Binder 是各种 Manager（ActivityManager、WindowManager 等）和相应 xxxManagerService 的桥梁；
5. 从 Android APP 层：Binder 是客户端和服务端进行通信的媒介，当 bindService 的时候，服务端会返回一个包含了服务端业务调用的 Binder 对象，通过这个 Binder 对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于 AIDL 的服务。

## 2.IPC原理

从进程角度来看IPC机制

![](http://gityuan.com/images/binder/prepare/binder_interprocess_communication.png)

每个 Android 的进程，只能运行在自己进程所拥有的虚拟地址空间。对应一个 4GB 的虚拟地址空间，其中 3GB 是用户空间，1GB 是内核空间，当然内核空间的大小是可以通过参数配置调整的。对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。Client 进程向 Server 进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，Client 端与 Server 端进程往往采用 ioctl 等方法跟内核空间的驱动进行交互。

## 3. Binder架构

![](http://gityuan.com/images/binder/java_binder/java_binder.jpg)

Binder 在整个 Android 系统中有这举足轻重的地位，在 Native 层有一套完整的 binder 通信的 C/S 架构(图中的蓝色)，Bpinder 作为客户端，BBinder 作为服务端。基于 naive 层的 Binder 框架，Java 也有一套镜像功能的 binder C/S 架构，通过 JNI 技术，与 native 层的 binder 对应，Java 层的 binder 功能最终都是交给 native 的 binder 来完成。从 kernel 到 native，jni，framework 层的架构所涉及的所有有关类和方法见 Binder 类图。[Binder类图](http://gityuan.com/2015/11/21/binder-framework/#binder-1)。

## 4. Binder原理

Binder 通信采用 C/S 架构，从组件视角来说，包含 Client、Server、ServiceManager 以及 binder 驱动，其中 ServiceManager 用于管理系统中的各种服务。架构图如下所示：

![](http://gityuan.com/images/binder/prepare/IPC-Binder.jpg)

ServiceManager

可以看出无论是注册服务和获取服务的过程都需要 ServiceManager，需要注意的是此处的 Service Manager 是指 Native 层的 ServiceManager（C++），并非指 framework 层的 ServiceManager(Java)。ServiceManager 是整个 Binder 通信机制的大管家，是 Android 进程间通信机制 Binder 的守护进程，要掌握 Binder 机制，首先需要了解系统是如何首次启动 Service Manager。当 Service Manager 启动之后，Client 端和 Server 端通信时都需要先获取 Service Manager 接口，才能开始通信服务。

图中 Client/Server/ServiceManage 之间的相互通信都是基于 Binder 机制。既然基于 Binder 机制通信，那么同样也是 C/S 架构，则图中的 3 大步骤都有相应的 Client 端与 Server 端。

1. **注册服务(addService)：**Server 进程要先注册 Service 到 ServiceManager。该过程：Server 是客户端，ServiceManager 是服务端。
2. **获取服务(getService)：**Client 进程使用某个 Service 前，须先向 ServiceManager 中获取相应的 Service。该过程：Client 是客户端，ServiceManager 是服务端。
3. **使用服务：**Client 根据得到的 Service 信息建立与 Service 所在的 Server 进程通信的通路，然后就可以直接与 Service 交互。该过程：client 是客户端，server 是服务端。

图中的 Client,Server,Service Manager 之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与 Binder 驱动进行交互的，从而实现 IPC 通信方式。其中 Binder 驱动位于内核空间，Client,Server,Service Manager 位于用户空间。Binder 驱动和 Service Manager 可以看做是 Android 平台的基础架构，而 Client 和 Server 是 Android 的应用层，开发人员只需自定义实现 client、Server 端，借助 Android 的基本平台架构便可以直接进行 IPC 通信。

## 4.1 以 startService 为例说明 Binder IPC 原理

Binder 通信采用 C/S 架构，从组件视角来说，包含 Client、Server、ServiceManager 以及 binder 驱动，其中 ServiceManager 用于管理系统中的各种服务。下面说说 startService 过程所涉及的 Binder 对象的架构图：

![](http://gityuan.com/images/binder/binder_start_service/ams_ipc.jpg)

1. **注册服务**：首先AMS注册到ServiceManager。该过程：AMS所在进程(system_server)是客户端，ServiceManager是服务端。
2. **获取服务**：Client进程使用AMS前，须先向ServiceManager中获取AMS的代理类AMP。该过程：AMP所在进程(app process)是客户端，ServiceManager是服务端。
3. **使用服务**： app进程根据得到的代理类AMP,便可以直接与AMS所在进程交互。该过程：AMP所在进程(app process)是客户端，AMS所在进程(system_server)是服务端。

这3大过程每一次都是一个完整的Binder IPC过程。

## 5.Binder路由- C/S模式

先来看看 Native Binder IPC 的两个重量级对象：BpBinder (客户端) 和 BBinder (服务端) 都是 Android 中 Binder 通信相关的代表，它们都从 IBinder 类中派生而来，关系图如下：



![](http://gityuan.com/images/binder/prepare/Ibinder_classes.jpg)

- client 端：BpBinder.transact() 来发送事务请求；
- server 端：BBinder.onTransact() 会接收到相应事务。



- IBinder 有一个重要方法 queryLocalInterface， 默认返回值为 NULL；
  - BBinder/BpBinder 都没有实现，默认返回 NULL；BnInterface 重写该方法；
  - BinderProxy(Java)默认返回 NULL；Binder(Java)重写该方法；
- IInterface 有一个重要方法 asBinder；
- IInterface 子类(服务端)会有一个方法 asInterface；

Native 层通过宏 IMPLEMENT_META_INTERFACE 来完成 asInterface 实现和 descriptor 的赋值过程；

对于 Java 层跟 Native 一样，也有完全对应的一套对象和方法:

- 例如 ActivityManagerNative， 通过实现 asInterface 方法，以及其通过其构造函数调用 attachInterface()，完成 descriptor 的赋值过程。
- 再如 AIDL 全自动生成 asInterface 和 descriptor 赋值过程。

同一个进程，请求 binder 服务，不需要创建 binder_ref，BpBinder 等这些对象，但是是否需要经过 binder call，取决于 descriptor 是否设置。 这就涉及到 Java 服务 Native 使用，或许 Native 服务在 Java 层使用，需要格外注意。

**binder 的路由原理**：

BpBinder 发送端，根据 handler，在当前 binder_proc 中，找到相应的 binder_ref，由 binder_ref 再找到目标 binder_node 实体，由目标 binder_node 再找到目标进程 binder_proc。简单地方式是直接把 binder_transaction 节点插入到 binder_proc 的 todo 队列中，完成传输过程。

对于 binder 驱动来说应尽可能地把 binder_transaction 节点插入到目标进程的某个线程的 todo 队列，效率更高。当 binder 驱动可以找到合适的线程，就会把 binder_transaction 节点插入到相应线程的 todo 队列中，如果找不到合适的线程，就把节点之间插入 binder_proc 的 todo 队列。

## BpBinder 和 BBinder关系

在 BpBinder 类的内部有一个成员变量 mHandle，它是一个 int 型变量。这个变量所代表的含义是什么呢？对于每一个经过 binder driver 传输的 BBinder 对象，binder driver 都会在驱动层为它构建一个 binder_node 数据结构；同时为这个 binder_node 生成一个“引用”：binder_ref，每一个 binder_ref 都有一个 int 型的描述符。BpBinder 的成员变量 mHandle 的值就是 bidner_ref 中的 int 型描述符，这样就建立起了用户层的 Bpbinder 和一个驱动空间的 binder_ref 数据结构的对应关系。通过 “BpBinder handle→binder_ref→binder_node→BBinder”这样的匹配关系，就可以建立一个 Bpbinder 对应一个 BBinder 的关系。 下面这张图描述了 Handle 和 binder_ref 以及 binder_node 的关系：

![](http://om4rextnc.bkt.clouddn.com/17-7-9/77242092.jpg)

一个 binder_node 可能有很多个 binder_ref 引用它，但是一个客户进程内，对于同一个 binder_node 的引用只会存在一份——也就是说，如果进程 A 中有一个 BBinder service，进程 B 持有多个 BpXXXService 的代理类，但是这些代理类都对应同一个 BpBinder 对象。这点其实在我们的“全家福”里面已经体现的很清楚了。BpRefBase 和 BpBinder 的关系不是继承关系，而是一个聚合关系。这点其实很好理解，对同一个 BBinder 维持多个 BpBinder 是一件很浪费空间的事情。另外，BpBinder 的数目也影响着 BBinder 的生命周期，同一个 BBinder 使用同一个 BpBinder 也简化了生命周期的管理。

上面我们讲到了通过 binder 的传输的 BBinder，驱动都会为之建立一个 binder_node 数据结构。那么一个 BBinder 是如何由一个进程传递到另一个进程，进而得到一个匹配的 BpBinder 的呢？这就不得不讲到 Parcel 了。



内部类 Stub 实际上就是一个 Binder 类，当客户端和服务端位于同一个进程时，方法调用不会走跨进程的 transact 过程，而位于不同进程时，方法调用会走 transact 过程，这个逻辑由 Stub 的内部代理类 Proxy 完成。

### asInterface

用于将服务端的 Binder 对象转化成客户端所需的 AIDL 接口类型的对象，这种转化是区分进程的，如果客户端和服务端处于同一进程，那么此方法返回的就是服务端的 Stub 对象本身，否则返回的是系统封装后的 Stub.proxy 对象。

### asBinder

用于返回当前 Binder 对象

### onTransact

此方法运行在服务端的 Binder 线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。

方法原型：

```java
pubic Boolean onTransact(int code, android.os.Parcel data,android.os.Parcel reply,int flags)
```

onTransact 方法执行过程：

1. 服务端通过 `code` 可以确定客户端请求的目标方法是什么
2. 接着从 `data` 中取出目标方法所需的参数（如果目标方法有参数的话），然后执行目标方法。
3. 当目标方法执行完后，就向 `reply` 中写入返回值（如果目标方法有返回值的话）

**注意：** 如果此方法返回 false，那么客户端请求会失败，因此可以利用这个特性来做**权限验证**。

0

<div class="tip">

1. 当客户端发起远程请求时，由于当前线程会被挂起直至服务端进程返回数据，所以如果一个远程方法很耗时，那么久不能再 UI 线程中发起此远程请求
2. 由于服务端 Binder 方法运行在 Binder 的线程池中，所以 Binder 方法不管是否耗时都应该采用同步的方式去实现

</div>



[Binder系列—开篇](http://gityuan.com/2015/10/31/binder-prepare/)

[Binder系列10—总结](http://gityuan.com/2015/11/28/binder-summary/)

[彻底理解Android Binder通信架构](http://gityuan.com/2016/09/04/binder-start-service/)

[Android Binder详解](https://mr-cao.gitbooks.io/android/content/android-binder.html#_bpbinder_bbinder)

[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)