# binder简介

## binder简介

同一个进程中函数之间只所以可以相互调用，是因为他们的虚拟地址是在同一个地址空间中的。他们知道相互的虚拟地址，所以进程中的函数才能相互调用。

那如何实现进程和进程之间的相互通信呢？

进程中0-3G工作在用户空间，而3G-4G是工作在内核状态，也就是说：**所有的进程共用3G-4G的空间**,既然所有进程都共用，那么我们就找到了进程通信的土壤。

借助土壤，就诞生的Linux基本的进程通信方式：

* fifo 和 pipe  
* signal
* message
* share memory
* semaphore
* socket

android 特有的方式：

* binder
* 双向管道



binder是Android特有的IPC（进程通信） RPC（跨进程调用）

## Android为啥使用binder作为进程间通信的基础：

### 传输性能

socket作为一款通用接口，其传输效率低，开销大，主要用在跨网络的进程间通信和本机上进程间的低速通信。消息队列和管道采用存储-转发方式，即数据先从发送方缓存区拷贝到内核开辟的缓存区中，然后再从内核缓存区拷贝到接收方缓存区，至少有两次拷贝过程。共享内存虽然无需拷贝，但控制复杂，难以使用。

| IPC                  | 数据拷贝次数 |
| -------------------- | ------------ |
| 共享内存             | 0            |
| Binder               | 1            |
| Socket/管道/消息队列 | 2            |

### 安全考虑

Android作为一个开放式，拥有众多开发者的平台，应用程序的来源广泛，确保智能终端的安全是非常重要的。

终端用户不希望从网上下载的程序在不知情的情况下偷窥隐私数据，连接无线网络，长期操作底层设备导致电池很快耗尽等等。传统IPC没有任何

安全措施，完全依赖上层协议来确保。首先传统IPC的接收方无法获得对方进程可靠的UID/PID（用户ID/进程ID），从而无法鉴别对方身份。

　　Android为每个安装好的应用程序分配了自己的UID，故进程的UID是鉴别进程身份的重要标志。使用传统IPC只能由用户在数据包里填入UID/PID，

但这样不可靠，容易被恶意程序利用。可靠的身份标记只有由IPC机制本身在内核中添加。其次传统IPC访问接入点是开放的，无法建立私有通道。

**比如命名管道的名称、system V的键值、socket的ip地址或文件名都是开放的**，只要知道这些接入点的程序都可以和对端建立连接，不管怎样都无法

阻止恶意程序通过猜测接收方地址获得连接。

　　基于以上原因，Android需要建立一套新的IPC机制来满足系统对通信方式，传输性能和安全性的要求，这就是Binder。

**Binder基于 Client-Server通信模式，传输过程只需一次拷贝，为发送方添加UID/PID身份，既支持实名Binder也支持匿名Binder，安全性高。**

### 面向对象的 Binder （在APP层）

　面向对象思想的引入将进程间通信转化为通过对某个Binder对象的引用调用该对象的方法，而其独特之处在于**Binder对象是一个可以跨进程引用的对象，它的实体位于一个进程中，而它的引用却遍布于系统的各个进程之中。**最诱人的是，这个引用和java里引用一样既可以是强类型，也可以是弱类型，而且可以从一个进程传给其它进程，让大家都能访问同一Server，就像将一个对象或引用赋

值给另一个引用一样。Binder模糊了进程边界，淡化了进程间通信过程，整个系统仿佛运行于同一个面向对象的程序之中。

　　面向对象只是针对应用程序而言，对于Binder驱动和内核其它模块一样使用C语言实现，没有类和对象的概念。

Binder驱动为面向对象的进程间通信提供底层支持。