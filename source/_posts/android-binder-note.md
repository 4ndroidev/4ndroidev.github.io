title: Android IPC机制
date: 2017-04-26 10:52:28
categories: Android
tags:
	- Android
	- 读书笔记
---

> 摘自 《Android 技术内幕》系统卷 -- 杨丰盛 著， 好记性不如烂笔头

## Binder概述

Linux系统中，以进程为单位分配和管理资源的。处于保护机制，一个进程不能直接访问另一个进程的资源，也就是说，进程间互相封闭。但是，一个复杂的应用系统中，通常会使用多个相关进程来共同完成一项任务，因此要求进程之间必须能够互相通信。从而共享资源和信息。所以，操作系统内核必须提供进程间的通信机制（IPC）。

应用程序虽然是以独立的进程来运行的，但是相互之间还是需要通信的，比如，在多进程的环境下，应用程序和后台服务通常会运行在不同的进程中，有着独立的地址空间，但是因为需要相互协助，彼此间又必须进行通信和数据共享，这就需要进程通信来完成。在Linux系统中，进程间通信的方式有socket、named pipe、message queue、signal、share memory等；Java系统中的进程间通信方式也有socket、named pipe等，所以Android可以选择的进程通信的方式也很多，但是主要包括以下几种方式

- 标准Linux Kernel IPC接口
- 标准D-BUS接口
- Binder接口

<!-- more -->

### 选择Binder的原因

因为Binder更加简洁和快速，消耗的内存资源更小，传统的进程间通信可能会增加进程的开销，而且有进程过载和安全漏洞等方面的风险，Binder正好能解决和避免这些问题。Binder主要能提供以下一些功能：

- 用驱动程序来推进进程间的通信。
- 通过共享内存来提高性能。
- 为进程请求分配每个进程的线程池。
- 针对系统中的对象引入了应用计数和跨进程的对象引用映射。
- 进程间同步调用

### 初识Binder

Binder是通过Linux的Binder Driver来实现的，Binder操作类似于线程迁移（thread migration）， 两个进程间通信看起来就像是一个进程进入另一个进程去执行代码，然后带着执行的结果返回。Binder的用空空间为每一个进程维护着一个可用的线程池，线程池用于处理到来的IPC以及执行进程的本地消息，Binder通信是同步的而不是异步的。同时，Binder机制是基于OpenBinder来实现的，是一个OpenBinder的Linux实现，Android系统的运行都将依赖Binder驱动。

Binder通信也是基于Service与Client的，所有需要IBinder通信的进程都必须创建一个IBinder接口。系统中有一个名为Service Manager的守护进程管理这系统中的各个服务，它负责监听是否有其他程序向其发送请求，如果有请求就响应，如果没有则继续监听等待。每个服务都要在Service Manager中注册，而请求服务的客户端则向Service Manager请求服务。在Android虚拟机启动之前，系统会先启动Service Manager进程，Service Manager就会打开Binder驱动，并通知Binder Kernel驱动程序，这个进程将作为System Service Manager，然后该进程将进入一个循环，等待处理来自其它进程的数据。因此，我们也可以将Binder的实现大致分为：Binder驱动、Service Manager、Service、Client这几部分，下面将分别对这几个部分进行详细分析。

## Binder驱动的原理和实现

任何上层应用程序接口和用户操作都需要底层硬件设备驱动的支持，并未起提供各种操作接口。

### Binder驱动的原理

为了完成进程间通信，Binder采用了AIDL（Android Interface Definition Language）来描述进程间的接口。在实际的实现中，Binder是作为一个特殊的字符型设备而存在的，设备节点为/dev/binder， 其实现遵循Linux设备驱动模型，实现代码主要涉及以下文件：

- kernel/drivers/staging/binder.h
- kernel/drivers/staging/binder.c

在其驱动的实现过程中，主要通过binder_ioctl函数与用户控件的进程交换数据。BINDER_WRITE_READ用来读写数据，数据包中有一个cmd域用于区分不同的请求。binder_thread_write函数用于发送请求和返回结果，而binder_thread_read函数则用于读取结果。在binder_thread_write函数中调用binder_transaction函数来转发请求并返回结果。当收到请求是，binder_transaction函数会通过对象的handle找到对象所在的进程，如果handle为空，就认为对象是context_mgr，把请求发给context_mgr所在的进程。请求中所有的Binder对象全部放在一个RB树中，最后把请求放在目标进程的队列中，等待目标进程读取。数据的解析工作放在binder_parse中实现；关于如何生成context_mgr，内核中提供了BINDER_SET_CONTEXT_MGR命令来完成此项功能。下面我们就来看看Binder驱动究竟是如何实现的。

### Binder驱动的实现

上面我们已经对Binder驱动的原理进行了分析，在开始分析驱动的实现之前，我们还是通过一个例子来说明Binder在实际应用中如何运用，以及它能帮我们解决什么样的问题。这样会更容易帮组大家理解Binder驱动的实现。比如，A进程如果要使用B进程的服务，B进程首先要注册此服务，A进程通过Binder获取该服务的handle，通过这个handle，A进程就可以使用该服务了。此外，你可以把handle理解成地址。A进程使用B进程的服务还意味着二者遵循相同的协议，这个协议反映在代码上就是二者都实现了IBinder接口。

Binder不仅是Android系统中的一个完善的IPC机制，它也可以被当做Android系统的一种RPC(远程过程调用)机制，因为Binder的功能就是在本地“执行”其他进程的功能。因此，进程通过Binder获取将要调用的进程服务时，可以是一个本地对象，也可以是一个远程服务的“引用”。

Binder的实质就是要把对从一个进程映射到另外一个进程中，而不管这个对象是本地的还是远程的。如果是本地对象，更好理解，如果是远程对象，即将远程对象的“引用”从一个进程映射到另一个进程中，于是当使用这个远程对象时，实际就是使用远程对象在本地的一个“引用”，类似于把这个远程对象当做一个本地对象在使用。这也就是Binder与其他IPC机制不同的地方。

这个本地“对象”与远程对象的“引用”有什么不同？本地“对象”表示本地进程的地址空间的一个地址，而远程对象的“引用”则是一个抽象的32位句柄。它们之间是互斥的：所有的进程本地对象都是本地进程的一个地址（address、ptr、binder），所有的远程进程的对象的“引用”都是一个句柄。对于发送者进程来说，不管是“对象”还是“引用”，它都会认为被发送的Binder对象是一个远程对象的句柄（即远程对象的“引用”）。当时，当Binder对象的数据被发送到远端接收进程时，远端接收进程则会被认为该Binder对象是一个本地对象地址（几本地对象）。正如我们之前说的，当Binder对象被接收进程接收后，不管该Binder对象是本地还是远程的，它都会被当作一个本地进程来处理。因此，从第三方的角度来说，尽管名称不同，对于一次完整的Binder调用，都将指向同一个对象，Binder驱动则负责两种不同名称的对象的正确映射，这样才能把数据发送给正确的进程进行通信。在本地则是地址（即本地对象的地址）。用对象的基础，对一个对象的引用，在远程是句柄，在本地则是地址（即本地对象的地址）。

## Binder的架构与实现

上一节分析了Android中的IPC机制--Binder驱动的实现。我们知道Binder在Android中占据着重要地位，需要完成进程之间通信的所有应用程序都使用了Binder机制，所以Android也对Binder驱动所提供的接口进行了封装，同时还在Android的工具库中提供了一套Binder库。

### Binder的系统架构

在分析Binder的系统架构之前，首先距离说明Binder的用处。在Android的设计中，`每个Activity都是一个独立的进程，每个Service也是一个独立的进程，而Activty要与Service进行通信，就是夸进程通信，这是就需要使用Binder机制了`（书本这里描述貌似有问题，可能是个假设，可能是我理解不到）。这里可以把Activity看作一个客户端，把Service看做一个服务端，实际上也是一个客户端与服务器之间的通信，具体的通信流程由Binder来完成。

#### Binder机制的组成

Android的Binder机制就是一个C/S架构，客户端和服务器直接通过Binder交互数据，打开Binder写入数据，通过Binder读取数据，这样通讯就可以完成了。数据的读写是由上一节所介绍的Binder驱动完成的，除了Binder驱动外，整个机制还包括以下几个组成部分：

1. Service Manager， 主要负责管理Android系统中所有的服务，当客户端要与服务端进行通信时，首先就会通过Service Manager来查询和取得所需要交互的服务。当然， 每个服务也都需要向Service Manager注册自己提供的服务，以便能提供客户端进行查询和获取。

2. 服务（Server）， 这里的服务即上面所说的服务端，通常也是Android的系统服务，通过Service Manager可以查询和获取某个Server。

3. 客户端（Client），这里的客户端一般是指Android系统上面的应用程序。它可以请求Server中的服务，比如Activity

4. 服务代理， 服务代理是指在客户端应用程序中生成的Server代理（proxy）。从应用程序的角度来看，代理对象和本地对象没有区别，都可以调用其方法，方法都是同步的，并且返回相应的结果。服务代理也是Binder机制的核心模块。

#### Binder的机制和原理

作为Android系统的核心机制，Binder几乎贯穿整个Android系统，本节将从Binder所涉及的Service Manager、服务、客户端、服务端（代理对象）等各个部分进行分析。Binder工作流程：

1. 客户端首先获取服务器的代理对象。 所谓的代理对象实际上就是在客户端建立一个服务端“引用”，该代理对象具有服务端的功能，使其在客户端访问服务端的方法就想访问本地方法一样。

2. 客户端通过调用服务器代理对象的方式向服务器端发送请求。

3. 代理对象将用户请求通过Binder驱动发送到服务器进程。

4. 服务器进程处理用户请求，并通过Binder驱动返回处理结果给客户端的服务器代理对象。

5. 客户端收到服务器端的返回结果。

