## Binder机制详解

## 概述

什么是`Binder`？`Binder`是Android程序中实现跨进程通信(**IPC**)的一种方式。指两个进程之间进行数据交换的过程

为什么要跨进程通信？

因为有**进程隔离**，进程隔离指的是，一个进程不能直接操作或者访问另一个进程。也就是进程A不可以直接访问进程B的数据。

那么如何进行跨进程通信呢？我们都知道，Android系统的内核是Linux，所以我们首先了解一下Linux上是如何实现的把。

## Linux上的跨进程通信机制

在Linux中有这么几种IPC机制，有管道（pipe）、信号（sinal）、消息队列（Message）、共享内存（Share Memory)、套接字（Socket）等。

- **管道** 管道是Linux从Unix中继承而来的进程间通信机制，管道本质上就是一个文件，不过是内存中的文件，前面的进程以写方式打开文件，后面的进程以读方式打开。这样前面写完后面读，于是就实现了通信。Linux上的管道就是一个操作方式为文件的内存缓冲区。管道采用的是半双工通信方式的，数据只能在一个方向上流动。

  

  ![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3aad1a69bdb142e29ecf170782aac2e8~tplv-k3u1fbpfcp-zoom-1.image)

  

- **信号** 信号(signal)是一种软中断，信号机制是进程间通信的一种方式，采用异步通信方式。在原理上，一个进程收到一个信号与处理器收到一个中断请求可以说是一样的。信号是异步的，一个进程不必通过任何操作来等待信号的到达，事实上，进程也不知道信号到底什么时候到达。

- **消息队列** 消息队列，就是一个消息的链表，是一系列保存在内核中消息的列表。用户进程可以向消息队列添加消息，也可以向消息队列读取消息。信息会复制两次。

- **共享内存** 顾名思义，共享内存就是允许两个不相关的进程访问同一个逻辑内存。共享内存是在两个正在运行的进程之间共享和传递数据的一种非常有效的方式。不同进程之间共享的内存通常安排为同一段物理内存

- **套接字** 套接字是更为基础的进程间通信机制，与其他方式不同的是，套接字可用于不同机器之间的进程间通信。

## Linux上IPC原理

首先我们需要理解**User space（用户空间）**和 **Kernel space（内核空间）**。

为了保护用户进程不能直接操作内核，保证内核的安全，**操作系统从逻辑上将虚拟空间划分为用户空间和内核空间**。Linux 操作系统将最高的1GB字节供内核使用，称为内核空间，较低的3GB 字节供各进程使用，称为用户空间。

内核空间是Linux内核的运行空间，用户空间是用户程序的运行空间。为了安全，它们是隔离的，即使用户的程序崩溃了，内核也不会受到影响。内核空间的数据是可以进程间共享的，而用户空间则不可以。

那万一用户空间需要用到内核空间的数据或者需要访问内核空间怎么办呢？那这就需要借助**系统调用**了。**系统调用是用户空间访问内核空间的唯一方式**，保证了所有的资源访问都是在内核的控制下进行的，避免了用户程序对系统资源的越权访问，提升了系统安全性和稳定性。

好，基本知识了解了，我们看一下Linux上IPC是怎么实现的。



![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b9778b9d7b741f4a44329df528e58aa~tplv-k3u1fbpfcp-zoom-1.image)



首先发送进程通过`copy_from_user`方法将发送的数据发送到内核缓存区，这期间会发生一次内存拷贝。

然后内核空间再通过`copy_to_user`方法将发送进程发送的数据传给接收进程，这期间会再一次发生内存拷贝。注意这两个方法是关键：

- `copy_from_user`：将用户空间的数据拷贝到内核空间。
- `copy_to_user`：将内核空间的数据拷贝到用户空间。

这就完成了一次跨进程通信，是不是感觉很简单？没错，确实不复杂，但是这个方式有两个问题，就是发送一次数据需要内存拷贝两次。第二个就是接收进程不知道发送进程要发送多大的数据，所以只能尽可能地往大了开辟内存或者事先在读取一次消息头来知晓数据大小，不是浪费空间就是浪费时间。那么Binder是怎么解决这个问题的呢？

## Binder原理



![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/023f8bbb1edf4dd785d523636a85bbf0~tplv-k3u1fbpfcp-zoom-1.image)

emmmm看不懂，什么是映射？这都是啥?





![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d39262d443ed4e62969a1a80393e3f7a~tplv-k3u1fbpfcp-zoom-1.image)



先来讲一下这个映射，什么是映射呢？映射就是**内存映射（mmap）**，是一种内存映射文件的方法，即将一个文件或者其他对象映射到进程的地址空间，实现文件磁盘地址和应用程序进程虚拟地址空间中一段虚拟地址的一一映射关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用read,write等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。

说得好，不过什么意思？

简单地说就是存在映射关系的双方，只要修改其中一方的内容，另一方也会发生改变。这个实现的手段是通过`mmap`方法来实现的。

**Binder没有物理介质，Binder是基于内存映射来实现的。**Binder这么做目的就是为了跨进程通信。如上图

1. Binder首先在内核空间创建了一个内核缓存区，然后又创建了一个数据接收缓存区。然后将内核缓存区和数据接收缓存区进行映射。
2. Binder再将接收缓存区和接收进程的用户空间地址建立映射关系。
3. 发送进程通过`copy_from_user`发送了数据到内核缓存区，这里进行了一次内存拷贝。由于映射关系的存在，数据立马被同步到了接收进程的内存空间。完成跨进程通信。整个过程只进行了一次内存拷贝。

哇塞，看起来好像是这样啊。Google果然是牛X，这么好的方法都能想得到，Linux发展这么多年怎么搞不出来Binder呢？其实也不是，Linux上也很早就有Binder了，Android系统之所以采用Binder机制来进行跨进程通信是有原因的。

## Android为什么使用Binder

1. **性能上**  Binder内存拷贝只需要一次，除了共享内存一次也不需要以外，Binder的性能是最高的。
2. **稳定性**   Binder是基于C/S架构的，这个架构通常采用两层结构，在技术上已经很成熟了，稳定性是没有问题的。共享内存没有分层，难以控制，并发同步访问临界资源时，可能还会产生死锁。从稳定性的角度讲，Binder是优于共享内存的。
3. **安全**  传统Linux IPC的接收方无法获得对方进程可靠的UID/PID，从而无法鉴别对方身份；而Android作为一个开放的开源体系，拥有非常多的开发平台，App来源甚广，因此手机的安全显得额外重要；对于普通用户，绝不希望从App商店下载偷窥隐射数据、后台造成手机耗电等等问题，传统Linux IPC无任何保护措施，完全由上层协议来确保。**Android系统中对外只暴露Client端，Client端将任务发送给Server端，Server端会根据权限控制策略，判断UID/PID是否满足访问权限，目前权限控制很多时候是通过弹出权限询问对话框，让用户选择是否运行**。
4. **语言**  Linux是基于C语言(面向过程的语言)，而Android是基于Java语言(面向对象的语句)，而对于Binder恰恰也符合面向对象的思想，将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中
5. **开源协议**  Linux内核是开源的系统，所开放源代码许可协议GPL保护。该协议具有“病毒式感染”的能力，只要进行了系统调用，调用到内核，那么也必须遵守GPL开源协议。Android 之父 Andy Rubin对于GPL显然是不能接受的，为此，Google巧妙地将GPL协议控制在内核空间，将用户空间的协议采用Apache-2.0协议。同时在GPL协议与Apache-2.0之间的Lib库中采用BSD证授权方法，有效隔断了GPL的传染性

## Binder作用

好吧，你说的很多确实很有道理，可是`Binder`到底有什么作用呢，我日常开发也没用到`Binder` 啊。

不不不，你错了，你的日常开发肯定用到了`Binder`，只不过你不知道而已。就以最常见的`startActivity`来说吧，经过层层调用，最后还是通过`Binder`来实现的。所以学习好`Binder`原理，更有利于我们了解Android系统的构建，也能开发出更高质量的APP。也是我们迈向高级Android开发工程师的必经之路~

## Binder架构设计

Binder架构也是采用分层架构设计, 每一层都有其不同的功能。分为`Java Binder`，`Native Binder`，`Kernel Binder` 。

Binder进程间通信采用的是C/S架构，也就是常说的客户端，服务端模式。这是一个简化版的示意图



![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00493ad2517a4811ac96f388587ba31e~tplv-k3u1fbpfcp-zoom-1.image)



可以看到的是，除了clien端和serve端，还多出了一个`ServiceManager`。`ServiceManager`用于管理系统中的各种服务。首先，Server需要向`ServiceManager`中注册服务，然后client如果想要使用Server提供的服务，需要向`ServiceManager`中查询，查询到了就可以使用服务。需要注意的是，上面这个简化图中client并不能直接使用服务，而是通过Binder跨进程通信的方式间接的使用服务。

这是一个稍微复杂一点的图：



![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6947068ed8af4cf69ad8b59a24ccb0b3~tplv-k3u1fbpfcp-zoom-1.image)



Client端和Server端直接全部都是通过虚线来连接的，也就表示虽然有调用关系，但是并不是直接调用。而是通过Binder实现的间接调用。需要注意的是此处的`Service Manager`是指Native层的`ServiceManager`（C++），并非指framework层的`ServiceManager`(Java)。

Client/Server/ServiceManage之间的相互通信都是基于Binder机制。既然基于Binder机制通信，那么同样也是C/S架构，则图中的3大步骤都有相应的Client端与Server端。

1. **注册服务**：首先Server注册到`ServiceManager`。该过程：Server所在进程是客户端，`ServiceManager`是服务端。
2. **获取服务**：Client进程使用Server前，须先向`ServiceManager`中获取Server的代理类BpBinder。该过程：BpBinder所在进程(app process)是客户端，`ServiceManager`是服务端。
3. **使用服务**： app进程根据得到的代理类`BpBinder`,便可以直接与Server所在进程交互。该过程：`BpBinder`所在进程(app process)是客户端，BBiner所在进程(system_server)是服务端。

这3大过程每一次都是一个完整的Binder IPC过程。

既然第一步是注册服务，那我们先从注册服务开始看起，看看服务到底是如何注册到系统中的。

## 服务注册

以MediaServer为例，看看它是如何注册的。为了方便阅读，全部剔除了Log等代码。

首先是MediaServer的入口函数：

```c++
//frameworks/av/media/mediaserver/main_mediaserver.cpp
int main(int argc __unused, char **argv __unused)
{
    signal(SIGPIPE, SIG_IGN);
    //获取ProcessState实例
    sp<ProcessState> proc(ProcessState::self());
    //获取BpServiceManager对象
    sp<IServiceManager> sm(defaultServiceManager());
    
    InitializeIcuOrDie();
    //注册MediaPlayerService
    MediaPlayerService::instantiate();//1
    ResourceManagerService::instantiate();
    registerExtensions();
    //启动Binder线程池
    ProcessState::self()->startThreadPool();
    //当前线程加入到线程池
    IPCThreadState::self()->joinThreadPool();
}
复制代码
```

第二行代码是获取`ProcessState`，`ProcessState`是何许人也？`ProcessState`从名称就可以看出来，用于代表进程的状态

我们看一下`ProcessState::self()`这个方法：

```c++
//frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState("/dev/binder");//1
    return gProcess;
}
复制代码
```

这是一个单例模式用于获取`ProcessState`，每个进程只有一个。眼尖的小伙伴应该发现了一个熟悉的字符串，没错就是这个`/dev/binder`，这就是没有物理介质的Binder驱动。那么`ProcessState`的构造方法又做了哪些事情呢？

#### ProcessState

```c++
//frameworks/native/libs/binder/ProcessState.cpp
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))//1 这一行很重要，打开了Binder驱动
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        //2 mmap内存映射
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}
复制代码
```

注释1处，`ProcessState`的构造方法首先调用了`open_driver()`方法，这个方法一看名字就知道，是打开驱动嘛，它的参数不就是我们刚才看到的传进去的`/dev/binder`，也就是说`ProcessState`在构造方法处打开了`Binder`驱动。看看它是怎么打开驱动的吧

##### open_driver

```c++
//frameworks/native/libs/binder/ProcessState.cpp
static int open_driver(const char *driver)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC);//1
    if (fd >= 0) {
        ...
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);//2
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '%s' failed: %s\n", driver, strerror(errno));
    }
    return fd;
}
复制代码
```

注释1处用于打开/dev/binder设备并返回文件操作符fd，这样就可以操作内核的Binder驱动了。

> Linux 系统中，把一切都看做是文件，当进程打开现有文件或创建新文件时，内核向进程返回一个文件描述符，文件描述符就是内核为了高效管理已被打开的文件所创建的索引，用来指向被打开的文件，所有执行I/O操作的系统调用都会通过文件描述符。

注释2处的ioctl函数的作用就是和Binder设备进行参数的传递，这里的ioctl函数用于设定binder支持的最大线程数为15（maxThreads的值为15）。

> 在用户空间，使用**ioctl方法系统调用来控制设备**。这是方法原型：
>
> ```c
> /*
> fd:文件描述符
> cmd:控制命令
> ...:可选参数:插入*argp，具体内容依赖于cmd*/
> int ioctl(int fd,unsigned long cmd,...);
> 复制代码
> ```
>
> 用户程序所作的只是通过命令码告诉驱动程序它想做什么，至于**怎么解释这些命令和怎么实现这些命令，这都是驱动程序要做的事情**。所以在用户空间我们想做什么事情都是通过这个方法以命令的形式告诉Binder驱动，Binder驱动收到命令执行相应的操作。

##### mmap

在刚才的注释2处就是大名鼎鼎的内存映射。内存映射函数mmap，给binder分配一块虚拟地址空间。它会在内核虚拟地址空间中申请一块与用户虚拟内存相同大小的内存，然后再申请物理内存，将同一块物理内存分别映射到内核虚拟地址空间和用户虚拟内存空间，实现了内核虚拟地址空间和用户虚拟内存空间的数据同步操作。

这是函数原型：

```c++
//原型
/*
addr: 代表映射到进程地址空间的起始地址，当值等于0则由内核选择合适地址，此处为0；
size: 代表需要映射的内存地址空间的大小，此处为1M-8K；
prot: 代表内存映射区的读写等属性值，此处为PROT_READ(可读取);
flags: 标志位，此处为MAP_PRIVATE(私有映射，多进程间不共享内容的改变)和 MAP_NORESERVE(不保留交换空间)
fd: 代表mmap所关联的文件描述符，此处为mDriverFD；
offset：偏移量，此处为0。

此处 mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
*/
void* mmap(void* addr, size_t size, int prot, int flags, int fd, off_t offset) 
复制代码
```

总的来说，`ProcessState`做了两件事情，一是打开Binder驱动并返回文件操作符fd，二是通过mmap为Binder分配了一块虚拟内存空间，以达到内存映射的目的。

#### MediaPlayerService::instantiate

这个方法呢就是注册服务：

```c++
void MediaPlayerService::instantiate() {
    //注册服务
    defaultServiceManager()->addService(String16("media.player"), new MediaPlayerService());
}
复制代码
```

`defaultServiceManager`返回的是`BpServiceManager`，这里先不讲`BpServiceManager`，你只需要知道`BpServiceManager`它实现了`IServiceManager`，并且通过内部有一个变量`mRemote= BpBinder`，`BpBinder`用来实现跨进程通信。 故此处等价于调用`BpServiceManager->addService`。

`BpBinder`是Client端与Server交互的代理类，而`BBinder`则代表了Server端。`BpBinder`和`BBinder`是一一对应的。`BpBinder`会通过`handle`来找到对应的`BBinder`。在`ServiceManager`中创建了`BpBinder`，通过`handle`(值为0)可以找到对应的`BBinder`。



![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b7377485bc048608a24b9164d592701~tplv-k3u1fbpfcp-zoom-1.image)



来看一下具体代码

```c++
//frameworks/native/libs/binder/IServiceManager.cpp
virtual status_t addService(const String16& name, const sp<IBinder>& service, bool allowIsolated) {
    Parcel data, reply; //Parcel是数据通信包
    //写入头信息"android.os.IServiceManager"
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());   
    data.writeString16(name);        // name为 "media.player"
    data.writeStrongBinder(service); // MediaPlayerService对象
    data.writeInt32(allowIsolated ? 1 : 0); // allowIsolated= false
    //remote()指向的是BpBinder对象
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR ? reply.readExceptionCode() : err;
}
复制代码
```

服务注册过程：向ServiceManager注册服务MediaPlayerService，服务名为”media.player”。addService函数的作用就是将请求数据打包成data，然后传给BpBinder的transact函数。

`data.writeStrongBinder(service)`这行代码中，将Binder对象进行了扁平化，转换成flat_binder_object对象。

- 对于Binder实体，则cookie记录Binder实体的指针；
- 对于Binder代理，则用handle记录Binder代理的句柄；(句柄这个翻译真的是脑残，你可以直接理解为ID或者编号)

##### BpBinder::transact

我们看一下transact函数:

```c++
// frameworks/native/libs/binder/BpBinder.cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
复制代码
```

`Binder`代理类调用`transact()`方法，真正工作还是交给`IPCThreadState`来进行`transact`工作。先来 看看`IPCThreadState::self`的过程。

##### IPCThreadState::self

```c++
//frameworks/native/libs/binder/IPCThreadState.cpp
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState;  //初始IPCThreadState 
    }

    if (gShutdown) return NULL;

    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) { //首次进入gHaveTLS为false
        if (pthread_key_create(&gTLS, threadDestructor) != 0) { //创建线程的TLS
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
复制代码
```

`TLS`是指Thread local storage(线程本地储存空间)，每个线程都拥有自己的TLS，并且是私有空间，线程之间不会共享。通过`pthread_getspecific/pthread_setspecific`函数可以获取/设置这些空间中的内容。从线程本地存储空间中获得保存在其中的IPCThreadState对象。这里可以得知`IPCThreadState::self()`实际上是为了创建`IPCThreadState`.

##### IPCThreadState

```c++
//frameworks/native/libs/binder/IPCThreadState.cpp
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mMyThreadId(gettid()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
复制代码
```

`pthread_setspecific`函数用于设置`TLS`，将`IPCThreadState::self()`获得的`TLS`和自身传进去。每个线程都有一个`IPCThreadState`，每个`IPCThreadState`中都有一个`mIn`、一个`mOut`,它们都是Parcel对象。成员变量`mProcess`保存了`ProcessState`变量(每个进程只有一个)。

- mIn 用来接收来自Binder设备的数据，默认大小为256字节；
- mOut用来存储发往Binder设备的数据，默认大小为256字节。

那这里构造完了，看一下`IPCThreadState`的`transact`函数：

##### IPCThreadState::transact

```c++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err = data.errorCheck(); //数据错误检查
    flags |= TF_ACCEPT_FDS;
    ....
    if (err == NO_ERROR) { // 传输数据
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    ...

    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            //等待响应 
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }

    } else {
        //oneway，则不需要等待reply的场景
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
复制代码
```

`IPCThreadState`进行`transact`事务处理分3部分：

- errorCheck() //数据错误检查
- writeTransactionData() // 传输数据
- waitForResponse() //等待响应

其中`writeTransactionData`函数用于传输数据，其中第一个参数`BC_TRANSACTION`代表向`Binder`驱动发送命令协议，向`Binder`设备发送的命令协议都以BC_开头，而`Binder`驱动返回的命令协议以BR开头。

我们就要看看到底是怎么写进去数据的：

##### IPCThreadState.writeTransactionData

```c++
//frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;
    tr.target.ptr = 0;
    tr.target.handle = handle; // handle = 0
    tr.code = code;            // code = ADD_SERVICE_TRANSACTION
    tr.flags = binderFlags;    // binderFlags = 0
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    // data为记录Media服务信息的Parcel对象
    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();  // mDataSize
        tr.data.ptr.buffer = data.ipcData(); //mData
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t); //mObjectsSize
        tr.data.ptr.offsets = data.ipcObjects(); //mObjects
    } else if (statusBuffer) {
        ...
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);         //cmd = BC_TRANSACTION
    mOut.write(&tr, sizeof(tr));  //写入binder_transaction_data数据
    return NO_ERROR;
}
复制代码
```

`binder_transaction_data`结构体是`binder`驱动通信的数据结构，该过程最终是把`Binder`请求码`BC_TRANSACTION`和`binder_transaction_data`结构体写入到`mOut`。其中`handle`的值用来标识目的端，注册服务过程的目的端为service manager，此处handle=0所对应的是`binder_context_mgr_node`对象，正是service manager所对应的binder实体对象。

transact方法中执行完writeTransactionData方法后，接下来就会执行waitForResponse()方法

##### IPCThreadState.waitForResponse

```c++
//frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;
    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;//1
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;
        cmd = (uint32_t)mIn.readInt32();
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;
       ...
        default:
            //处理各种命令协议
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
}
finish:
    ...
    return err;
}
复制代码
```

这里开启了一个死循环，然后从`talkWithDriver()`方法中发送并读取数据。在`waitForResponse`过程, 首先执行`BR_TRANSACTION_COMPLETE`；另外，目标进程也就是Server端收到事务后，处理`BR_TRANSACTION`事务。 然后发送给当前进程也就是Client端，再执行`BR_REPLY`命令。这是通信的具体流程：

![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)

Binder协议包含在IPC数据中，分为两类:

1. `BINDER_COMMAND_PROTOCOL`：binder请求码，以”BC_“开头，简称BC码，用于从IPC层传递到Binder Driver层；
2. `BINDER_RETURN_PROTOCOL` ：binder响应码，以”BR_“开头，简称BR码，用于从Binder Driver层传递到IPC层；

在这个过程中, 收到以下任一BR_命令，处理后便会退出waitForResponse()的状态:

- **BR_TRANSACTION_COMPLETE**: binder驱动收到BC_TRANSACTION事件后的应答消息; 对于oneway transaction,当收到该消息,则完成了本次Binder通信;
- **BR_DEAD_REPLY**: 回复失败，往往是线程或节点为空. 则结束本次通信Binder;
- **BR_FAILED_REPLY**:回复失败，往往是transaction出错导致. 则结束本次通信Binder;
- **BR_REPLY**: Binder驱动向Client端发送回应消息; 对于非oneway transaction时,当收到该消息,则完整地完成本次Binder通信;

**`talkWithDriver()`这个方法是真正的向内核发送消息的方法**。这是`talkWithDriver`：

##### IPCThreadState.talkWithDriver()

```c++
//frameworks/native/libs/binder/IPCThreadState.cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    ...
    //和Binder驱动通信的结构体    
    binder_write_read bwr;
    //mIn是否有可读的数据，接收的数据存储在mIn
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    //将要发送给Binder设备的消息填充到与Binder通信的结构体中
    bwr.write_buffer = (uintptr_t)mOut.data();
    
    if (doReceive && needRead) {
        //接收数据缓冲区信息的填充。如果以后收到数据，就直接填在mIn中了。
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    //当读缓冲和写缓冲都为空，则直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        //通过ioctl不停的读写操作，跟Binder Driver进行通信
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        ...
    } while (err == -EINTR); //当被中断，则继续执行
    ...
    return err;
}
复制代码
```

跟内核通讯都是通过ioctl方法进行的。在前面的介绍中已经讲解了这个方法的原型。binder_write_read结构体用来与Binder设备交换数据的结构, 通过ioctl与mDriverFD通信，是真正与Binder驱动进行数据读写交互的过程。 主要是操作mOut和mIn变量。

**binder_write_read是整个Binder IPC过程，最为核心的数据结构之一。**



![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)



看懵了？捋一下思路，数据的转变是这样的：

`IServiceManager.addService()`方法将数据打包为**Parcel**---->`IPCThreadState.writeTransactionData()`将**Parcel**打包为**binder_transaction_data**----->`IPCThreadState.talkWithDriver()`方法将**binder_transaction_data**打包为**binder_write_read**。

接下来就要进入到内核Binder中了。

#### Binder Driver

在了解Binder驱动之前，首先我们来了解一个概念，就是 **binder_procs**。什么是 **binder_procs**，**binder_procs是一个链表**。对于底层Binder驱动，通过 binder_procs 链表记录所有创建的 binder_proc 结构体，binder 驱动层的每一个 binder_proc 结构体都与用户空间的一个用于 binder 通信的进程一一对应。可以理解为，**一个binder_proc 结构体对应一个服务进程**，**而binder_procs记录了所有的binder_proc 结构体**。

每个进程有且只有一个 `ProcessState` 对象(上文中用来打开Binder驱动并分配内存的那个家伙)，这是通过单例模式来保证的。在每个进程中可以有很多个线程，每个线程对应一个 IPCThreadState(上文中用来发送通信数据给Binder驱动的那个家伙)对象，IPCThreadState 对象也是单例模式，即一个线程对应一个 IPCThreadState 对象，在 Binder 驱动层也有与之相对应的结构，那就是 Binder_thread 结构体。在 binder_proc 结构体中通过成员变量 rb_root threads，来记录当前进程内所有的 binder_thread。



![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5d7d03752e8431392178028efe87991~tplv-k3u1fbpfcp-zoom-1.image)



对于一个`binder_proc`，也就是一个服务进程来说。它有四个非常重要的结构体，分别是`binder_thread`，`binder_node`，`binder_ref`，`binder_buffer`。

- **Binder实体对象binder_node**，表示当前进程提供了哪些binder服务；

  > 可以将binder_proc理解为一个进程，而将binder_noder理解为一个Service。binder_proc里面有一个红黑树，用来保存所有在它所描述的进程里面创建Service。而每一个Service在Binder驱动里面都有一个binder_node来描述。

- **Binder引用对象binder_ref**，表示哪些进程持有当前进程的binder的引用；每一个Clinet在驱动中都有一个binder_ref和他对应，内部有对应的进程信息binder_proc和binder_node信息。client端拿着BpBinder引用对象，在驱动层对应着binder_ref，就可以找到binder_proc和binder_node信息，找到指定的进程可以实现交互。

- **Binder_thread**，进程在启动时会创建一个binder线程池，并向其中注册一个Binder线程；驱动发现没有空闲binder线程时会向Server进程注册新的的binder线程。对于一个Server进程有一个最大Binder线程数限制，默认为16个binder线程。对于所有Client端进程的binder请求都是交由Server端进程的binder线程来处理的。

- **binder_buffer** 内核缓冲区，用以在进程间传递数据。Binder驱动将Binder进程的内存分成一段一段进行管理，而这每一段则是使用binder_buffer来描述的。

上图是从结构体的角度来看待**binder_procs**，我们换个角度，从进程与线程的角度来看一下是什么样



![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd35310f686e42c3bdef8f971def1608~tplv-k3u1fbpfcp-zoom-1.image)



有的小伙伴会发现，在Binder_proc或者binder_thread中有个list_head todo这样的东西。这是个什么东西呢？list_head看名字就知道了是个头结点，todo呢？是要做的事情。没错，这个是用来保存要做的事情的队列。

在Binder驱动层，每个接收端进程都有一个todo队列，用于保存发送端进程发送过来的binder请求，这类请求可以由接收端进程的任意一个空闲的binder线程处理；接收端进程存在一个或多个binder线程，在每个binder线程里都有一个todo队列，也是用于保存发送端进程发送过来的binder请求，这类请求只能由当前binder线程来处理。也就是说**对于进程间的通信，就是发送端把binder_transaction节点，插入到目标进程或其子线程的todo队列中，等目标进程或线程不断循环地从todo队列中取出数据并进行相应的操作**。



![img](data:image/svg+xml;utf8,<?xml version="1.0"?><svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="800" height="600"></svg>)



那么接下来看一下内核中是怎么做的吧。

先看一下调用链:

> ioctl -> binder_ioctl -> binder_ioctl_write_read

##### binder_ioctl_write_read

```c
//binder.c
static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    struct binder_proc *proc = filp->private_data;
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    //将用户空间bwr结构体拷贝到内核空间 
    copy_from_user(&bwr, ubuf, sizeof(bwr));
    ...

    if (bwr.write_size > 0) {
        //将数据放入目标进程
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        ...
    }
    if (bwr.read_size > 0) {
        //读取自己队列的数据 
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
             bwr.read_size,
             &bwr.read_consumed,
             filp->f_flags & O_NONBLOCK);
        if (!list_empty(&proc->todo))
            wake_up_interruptible(&proc->wait);
        ...
    }

    //将内核空间bwr结构体拷贝到用户空间
    copy_to_user(ubuf, &bwr, sizeof(bwr));
    ...
}  
复制代码
```

接下来的内容有部分计划在后面的篇幅中讲解，这里就不在继续过多的说了。我大致说一下流程吧。

1. Binder 驱动收到请求命令向 ServiceManager 的 todo 队列里面添加一条注册服务的事务。事务的任务就是创建服务端进程 binder_node 信息并插入到 binder_procs 链表中。
2. 事务处理完之后发送 BR_TRANSACTION 命令，ServiceManager 收到命令后向 svcinfo 列表中添加已经注册的服务。svcinfo列表中记录着服务名和handle信息
3. 最后发送 BR_REPLY 命令唤醒等待的线程，通知注册成功

## 总结

用一张图来总结一下吧



![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7dcdc12e294d43e49a3af0cf06da3505~tplv-k3u1fbpfcp-zoom-1.image)



服务注册过程(addService)核心功能：在服务所在进程创建binder_node，在servicemanager进程创建binder_ref。要注意，此时要注册的进程也就是服务进程是Client端，而ServiceManager是server端。**每个进程binder_proc所记录的binder_ref的handle值是从1开始递增的；因为所有进程binder_proc所记录的handle=0的binder_ref都指向service manager；**

**整个过程copy once便是指binder_transaction()过程把binder_transaction_data->data拷贝到目标进程的buffer。**

需要了解的几个方法：

- binder_thread_write: 添加成员到todo队列;
- binder_thread_read: 消耗todo队列;

关于BpBinder和BBinder的对应，前面只是大概的讲了一下对应关系，这里详细的说明一下：



![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04e42931283943879ab86becbed51e60~tplv-k3u1fbpfcp-zoom-1.image)



比如进程1的BpBinder在发起跨进程调用时，向binder驱动传入了自己记录的句柄值，binder驱动就会在“进程1对应的binder_proc结构”的引用树中查找和句柄值相符的binder_ref节点，一旦找到binder_ref节点，就可以通过该节点的node域找到对应的binder_node节点，这个目标binder_node当然是从属于进程2的binder_proc啦，不过不要紧，因为binder_ref和binder_node都处于binder驱动的地址空间中，所以是可以用指针直接指向的。目标binder_node节点的cookie域，记录的其实是进程2中BBinder的地址，binder驱动只需把这个值反映给应用层，应用层就可以直接拿到BBinder了。

这就是服务注册的过程。从Native层到内核空间，流程就是这样喵~







转自：[掘金-Mlx](https://juejin.im/post/6867139592739356686#heading-6)

