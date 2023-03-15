## IO 模型简介

### 基础概念

在进行介绍之前先简单了解一些概念

#### 用户空间和内核空间

现在操作系统都是采用虚拟存储器，那么对 32 位操作系统而言，它的寻址空间（虚拟存储空间）为 4G（2 的 32 次方）。操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。针对 linux 操作系统而言，将最高的 1G 字节（从虚拟地址 0xC0000000 到 0xFFFFFFFF），供内核使用，称为内核空间，而将较低的 3G 字节（从虚拟地址 0x00000000 到 0xBFFFFFFF），供各个进程使用，称为用户空间。用户空间和内核空间

#### 进程切换

为了控制进程的执行，内核必须有能力挂起正在 CPU 上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。 进程切换很消耗资源。

#### 进程阻塞

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得 CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用 CPU 资源的。

#### 文件描述符

文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。有时也叫它文件句柄

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于 UNIX、Linux 这样的操作系统。

#### 缓存 I/O

缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

数据在传输过程中需要在应用程序地址空间和内核进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

### I/O 执行的两大阶段

在 Linux 中，对于一次 I/O 读取的操作，数据并不会直接拷贝到程序的程序缓冲区。通常包括两个不同阶段：

1. 等待数据准备好，到达内核空间 (Waiting for the data to be ready) ；
2. 从内核向进程复制数据 (Copying the data from the kernel to the process)
   对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。当所有等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到应用程序缓冲区。

### Linux 五大模型及对比

#### 同步阻塞 IO（blocking IO）

同步阻塞 IO 模型是最简单的 IO 模型，用户线程在内核进行 IO 操作时被阻塞

![](https://pic4.zhimg.com/80/v2-4e2522fa7b0d14530b08097821bf8daf_720w.webp)

![](https://pic3.zhimg.com/80/v2-8caf9a2d0a49c3d72cddb40de7f2979e_720w.webp)

用户线程通过系统调用 read 发起 IO 读操作，由用户空间转到内核空间。内核等到数据包到达后，然后将接收的数据拷贝到用户空间，完成 read 操作。

用户线程使用同步阻塞 IO 模型的伪代码描述为：

```c
{
    ...
    read(socket, buffer);
    process(buffer);
    ...
}
```

即用户需要等待 read 将 socket 中的数据读取到 buffer 后，才继续处理接收的数据。整个 IO 请求的过程中，用户线程是被阻塞的，这导致用户在发起 IO 请求时，不能做任何事情，对 CPU 的资源利用率不够。

#### 同步非阻塞 IO（nonblocking IO）

同步非阻塞 IO 是在同步阻塞 IO 的基础上，将 socket 设置为 NONBLOCK。这样做用户线程可以在发起 IO 请求后可以立即返回。

![](https://pic1.zhimg.com/v2-911e402697f97ffe8e5a5600c7270a3c_r.jpg)

![](https://pic2.zhimg.com/80/v2-6d7f042c99e0117df576b4e3e09f22f9_720w.webp)

由于 socket 是非阻塞的方式，因此用户线程发起 IO 请求时立即返回。但并未读取到任何数据，用户线程需要不断地发起 IO 请求，直到数据到达后，才真正读取到数据，继续执行。

用户线程使用同步非阻塞 IO 模型的伪代码描述为：

```c
{
    while(read(socket, buffer) != SUCCESS){
        // 轮询等待
    }
    process(buffer);
}
```

即用户需要不断地调用 read，尝试读取 socket 中的数据，直到读取成功后，才继续处理接收的数据。整个 IO 请求的过程中，虽然用户线程每次发起 IO 请求后可以立即返回，但是为了等到数据，仍需要不断地轮询、重复请求，消耗了大量的 CPU 的资源。一般很少直接使用这种模型，而是在其他 IO 模型中使用非阻塞 IO 这一特性。

#### IO 多路复用（IO multiplexing）

IO 多路复用模型是建立在内核提供的多路分离函数 select 基础之上的，使用 select 函数可以避免同步非阻塞 IO 模型中轮询等待的问题。

![[Pasted image 20230315235820.jpg]]
![[v2-fba6c2868dc5d8e1f369c09e785ace61_720w.webp]]
用户首先将需要进行 IO 操作的 socket 添加到 select 中，然后阻塞地等待 select 系统调用返回。当数据到达时，socket 被激活，select 函数返回。用户线程正式发起 read 请求，读取数据并继续执行。

从流程上来看，使用 select 函数进行 IO 请求和同步阻塞模型没有太大的区别，甚至还多了添加监视 socket，以及调用 select 函数的额外操作，效率更差。但是，使用 select 以后最大的优势是用户可以在一个线程内同时处理多个 socket 的 IO 请求。用户可以注册多个 socket，然后不断地调用 select 读取被激活的 socket，即可达到在同一个线程内同时处理多个 IO 请求的目的。而在同步阻塞模型中，必须通过多线程的方式才能达到这个目的。

用户线程使用 select 函数的伪代码描述为：

```c
{
    select(socket);
    while(1){
        sockets = select();
        for(socket in sockets) {
            if(can_read(socket)) {
                read(socket, buffer);
                process(buffer);
            }else if(can_write(socket)){
                write(socket, buffer);
                process(buffer);
            }else{
                // ....
            }
        }
    }
}
```

#### 信号驱动 IO（signal driven IO）

使用信号，让内核在文件描述符就绪的时候使用 SIGIO 信号来通知我们。我们将这种模式称为信号驱动 I/O 模式。
![[Pasted image 20230315235835.jpg]]

允许 Socket 使用信号驱动 I/O ，还要注册一个 SIGIO 的处理函数，这时的系统调用将会立即返回。然后我们的程序可以继续做其他的事情，当数据就绪时，进程收到系统发送一个 SIGIO 信号，可以在信号处理函数中调用 IO 操作函数处理数据

由于信号驱动 IO 在实际中并不常用，在此不做具体分析。

#### 异步 IO（asynchronous IO）

Linux 下的异步 IO 其实用得很少 ，著名的高性能网络框架 netty 5.0 版本被废弃的原因便是：使用异步 IO 提升效率，增加了复杂性，却并且没有显示出明显的性能优势。

第一阶段：当在异步 I/O 模型下时，用户进程如果想进行 I/O 操作，只需进行系统调用，告知内核要进行 I/O 操作，此时内核会马上返回， 用户进程 就可以去处理其他的逻辑了 。

第二阶段：当内核完成所有的 I/O 操作和数据拷贝后，内核将通知我们的程序，此时数据已经在用户空间了,可以对数据进行处理了

![[Pasted image 20230315235843.jpg]]

#### 五大 IO 模型的对比

![[v2-a9da26b83eb1011b230647977d16c8f7_720w.webp]]

前四种 I/O 模型都是同步 I/O 操作，他们的区别在于第一阶段，而他们的第二阶段是一样的。

在数据从内核复制到应用缓冲区期间（用户空间），进程阻塞于系统调用。相反，异步 I/O 模型在这等待数据和接收数据的这两个阶段里面都是非阻塞的，可以处理其他的逻辑，用户进程将整个 IO 操作交由内核完成，内核完成后会发送通知。在此期间，用户进程不需要去检查 IO 操作的状态，也不需要主动的去拷贝数据。

### 同步、异步、阻塞与非阻塞的重点概念区分

同步和异步的概念描述的是用户线程与内核的交互方式：同步是指用户线程发起 IO 请求后需要等待或者轮询内核 IO 操作完成后才能继续执行；而异步是指用户线程发起 IO 请求后仍继续执行，当内核 IO 操作完成后会通知用户线程，或者调用用户线程注册的回调函数。

阻塞和非阻塞的概念描述的是用户线程调用内核 IO 操作的方式：阻塞是指 IO 操作需要彻底完成后才返回到用户空间；而非阻塞是指 IO 操作被调用后立即返回给用户一个状态值，无需等到 IO 操作彻底完成

同步和异步、阻塞和非阻塞这两对儿概念，是两个维度的概念，大家不要混淆。

- 同步才区分阻塞和非阻塞
- 异步则一定是非阻塞的，不存在直接的异步非阻塞方式

## Reactor 模式和 Proactor 模式

> 上一章内容是本章内容的理论基础和底层依赖。本章内容则是在上章内容作为底层的基础，经过巧妙的设计和前赴后继的实践，得出的一套应用层的“最佳实践”。虽不是开箱即用，但也为我们提供了很大的便利，让我们少走很多弯路。下面我们就看看有哪些不错的架构模型、模式值得我们去参考。

在 web 服务中，处理 web 请求通常有两种体系结构，分别为：

- thread-based architecture（基于线程的架构）
- event-driven architecture（事件驱动模型）

### 基于线程的架构

thread-based architecture（基于线程的架构），通俗的说就是：多线程并发模式，一个连接一个线程，服务器每当收到客户端的一个请求， 便开启一个独立的线程来处理。

![](https://pic1.zhimg.com/80/v2-288b2a61dbfcf488eefd4a6ab9ad08dc_720w.webp)

这种模式一定程度上极大地提高了服务器的吞吐量，由于在不同线程中，之前的请求在 read 阻塞以后，不会影响到后续的请求。**但是**，仅适用于于并发量不大的场景，因为：

- 线程需要占用一定的内存资源
- 创建和销毁线程也需一定的代价
- 操作系统在切换线程也需要一定的开销
- 线程处理 I/O，在等待输入或输出的这段时间处于空闲的状态，同样也会造成 cpu 资源的浪费

**如果连接数太高，系统将无法承受**

### 事件驱动模型

事件驱动体系结构是目前比较广泛使用的一种。这种方式会定义一系列的事件处理器来响应事件的发生，并且将**服务端接受连接**与**对事件的处理**分离。其中，**事件是一种状态的改变**。比如，tcp 中 socket 的 new incoming connection、ready for read、ready for write。

如果对 event-driven architecture 有深入兴趣，可以看下维基百科对它的解释：[传送门](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Event-driven_architecture)

Reactor 模式和 Proactor 模式都是是 event-driven architecture（事件驱动模型）的实现方式，下面聊一聊这两种模式。

#### Reactor 模式

维基百科对`Reactor pattern`的解释：

> The reactor design pattern is an event handling pattern for handling service requests delivered concurrently to a service handler by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to the associated request handlers

从这个描述中，我们知道 Reactor 模式**首先是事件驱动的，有一个或多个并发输入源，有一个 Service Handler，有多个 Request Handlers**；Service Handler 会对输入的请求（Event）进行多路复用，并同步地将它们分发给相应的 Request Handler。

下面的图将直观地展示上述文字描述：

![](https://pic3.zhimg.com/80/v2-cfd7ed4a76c7b1df386bc4bd9576faca_720w.webp)

Reactor 模式也有三种不同的方式，下面一一介绍。

##### 单线程模式

Java 中的 NIO 模式的 Selector 网络通讯，其实就是一个简单的 Reactor 模型。可以说是单线程的 Reactor 模式

![](https://pic4.zhimg.com/80/v2-5f97abecc66698d6b1ce3034267e1fff_720w.webp)

Reactor 的单线程模式的单线程主要是针对于 I/O 操作而言，也就是所以的 I/O 的 accept()、read()、write()以及 connect()操作都在一个线程上完成的。

但在目前的单线程 Reactor 模式中，不仅 I/O 操作在该 Reactor 线程上，连非 I/O 的业务操作也在该线程上进行处理了，这可能会大大延迟 I/O 请求的响应。所以我们应该将非 I/O 的业务逻辑操作从 Reactor 线程上卸载，以此来加速 Reactor 线程对 I/O 请求的响应。

##### 工作者线程池模式

与单线程模式不同的是，添加了一个**工作者线程池**，并将非 I/O 操作从 Reactor 线程中移出转交给工作者线程池（Thread Pool）来执行。这样能够提高 Reactor 线程的 I/O 响应，不至于因为一些耗时的业务逻辑而延迟对后面 I/O 请求的处理。

![](https://pic3.zhimg.com/80/v2-2ffa44b686eea3ce55c7489fd67d1c1e_720w.webp)

在工作者线程池模式中，虽然非 I/O 操作交给了线程池来处理，但是**所有的 I/O 操作依然由 Reactor 单线程执行**，在高负载、高并发或大数据量的应用场景，依然较容易成为瓶颈。所以，对于 Reactor 的优化，又产生出下面的多线程模式。

##### 多线程模式

对于多个 CPU 的机器，为充分利用系统资源，将 Reactor 拆分为两部分：mainReactor 和 subReactor

![](https://pic4.zhimg.com/80/v2-14b10c1dd4c45a1fe3fd92f91fffe2e3_720w.webp)

**mainReactor**负责监听 server socket，用来处理网络新连接的建立，将建立的 socketChannel 指定注册给 subReactor，通常**一个线程**就可以处理 ；

**subReactor**维护自己的 selector, 基于 mainReactor 注册的 socketChannel 多路分离 I/O 读写事件，读写网络数据，通常使用**多线程**；

对非 I/O 的操作，依然转交给工作者线程池（Thread Pool）执行。

此种模型中，每个模块的工作更加专一，耦合度更低，性能和稳定性也大量的提升，支持的可并发客户端数量可达到上百万级别。关于此种模型的应用，目前有很多优秀的框架已经在应用了，比如 mina 和 netty 等。Reactor 模式-多线程模式下去掉工作者线程池（Thread Pool），则是 Netty 中 NIO 的默认模式。

- mainReactor 对应 Netty 中配置的 BossGroup 线程组，主要负责接受客户端连接的建立。一般只暴露一个服务端口，BossGroup 线程组一般一个线程工作即可
- subReactor 对应 Netty 中配置的 WorkerGroup 线程组，BossGroup 线程组接受并建立完客户端的连接后，将网络 socket 转交给 WorkerGroup 线程组，然后在 WorkerGroup 线程组内选择一个线程，进行 I/O 的处理。WorkerGroup 线程组主要处理 I/O，一般设置`2*CPU核数`个线程

#### Proactor 模式

流程与 Reactor 模式类似，区别在于 proactor 在 IO ready 事件触发后，完成 IO 操作再通知应用回调。虽然在 linux 平台还是基于 epoll/select，但是内部实现了异步操作处理器(Asynchronous Operation Processor)以及异步事件分离器(Asynchronous Event Demultiplexer)将 IO 操作与应用回调隔离。经典应用例如 boost asio 异步 IO 库的结构和流程图如下：

![](https://pic4.zhimg.com/80/v2-3ed3d63b31460c562e43dfd32d808e9b_720w.webp)

再直观一点，就是下面这幅图：

![](https://pic1.zhimg.com/80/v2-ae0c50cb3b3480fc36b8614b8b77f528_720w.webp)

再再直观一点，其实就回到了五大模型-异步 I/O 模型的流程，就是下面这幅图：

![](https://pic3.zhimg.com/80/v2-557eee325d2e29665930825618f7b212_720w.webp)

针对第二幅图在稍作解释：

Reactor 模式中，用户线程通过向 Reactor 对象注册感兴趣的事件监听，然后事件触发时调用事件处理函数。而 Proactor 模式中，用户线程将 AsynchronousOperation（读/写等）、Proactor 以及操作完成时的 CompletionHandler 注册到 AsynchronousOperationProcessor。

AsynchronousOperationProcessor 使用 Facade 模式提供了一组异步操作 API（读/写等）供用户使用，当用户线程调用异步 API 后，便继续执行自己的任务。AsynchronousOperationProcessor 会开启独立的内核线程执行异步操作，实现真正的异步。当异步 IO 操作完成时，AsynchronousOperationProcessor 将用户线程与 AsynchronousOperation 一起注册的 Proactor 和 CompletionHandler 取出，然后将 CompletionHandler 与 IO 操作的结果数据一起转发给 Proactor，Proactor 负责回调每一个异步操作的事件完成处理函数 handle_event。虽然 Proactor 模式中每个异步操作都可以绑定一个 Proactor 对象，但是一般在操作系统中，Proactor 被实现为 Singleton 模式，以便于集中化分发操作完成事件。

#### 总结对比

##### 主动和被动

以主动写为例：

- Reactor 将 handler 放到 select()，等待可写就绪，然后调用 write()写入数据；写完数据后再处理后续逻辑；
- Proactor 调用 aoi_write 后立刻返回，由内核负责写操作，写完后调用相应的回调函数处理后续逻辑

**Reactor 模式是一种被动的处理**，即有事件发生时被动处理。而**Proator 模式则是主动发起异步调用**，然后循环检测完成事件。

##### 实现

Reactor 实现了一个被动的事件分离和分发模型，服务等待请求事件的到来，再通过不受间断的同步处理事件，从而做出反应；

Proactor 实现了一个主动的事件分离和分发模型；这种设计允许多个任务并发的执行，从而提高吞吐量。

所以涉及到文件 I/O 或耗时 I/O 可以使用 Proactor 模式，或使用多线程模拟实现异步 I/O 的方式。

##### 优点

Reactor 实现相对简单，对于链接多，但耗时短的处理场景高效；

- 操作系统可以在多个事件源上等待，并且避免了线程切换的性能开销和编程复杂性；
- 事件的串行化对应用是透明的，可以顺序的同步执行而不需要加锁；
- 事务分离：将与应用无关的多路复用、分配机制和与应用相关的回调函数分离开来。

Proactor 在**理论上**性能更高，能够处理耗时长的并发场景。为什么说在**理论上**？请自行搜索 Netty 5.X 版本废弃的原因。

##### 缺点

Reactor 处理耗时长的操作会造成事件分发的阻塞，影响到后续事件的处理；

Proactor 实现逻辑复杂；依赖操作系统对异步的支持，目前实现了纯异步操作的操作系统少，实现优秀的如 windows IOCP，但由于其 windows 系统用于服务器的局限性，目前应用范围较小；而 Unix/Linux 系统对纯异步的支持有限，应用事件驱动的主流还是通过 select/epoll 来实现。

##### 适用场景

Reactor：同时接收多个服务请求，并且依次同步的处理它们的事件驱动程序；

Proactor：异步接收和同时处理多个服务请求的事件驱动程序。

## Select、Poll、Epoll 机制

> 本章（第三章）内容其实和第二章内容，都是第一张内容的延伸。第二章内容是第一章内容的延伸，本章内容则是第一章内容再往底层方面的延伸，也是面试中考察网络方面知识时，可能会问到的几个点。

select、poll、epoll 都是 I/O 多路复用的机制。I/O 多路复用就是通过一种机制，一个进程可以监视多个文件描述符，一旦某个描述符就绪（读就绪或写就绪），能够通知程序进行相应的读写操作 。

但是，select，poll，epoll 本质还是同步 I/O（I/O 多路复用本身就是同步 IO）的范畴，因为它们都需要在读写事件就绪后线程自己进行读写，读写的过程阻塞的。而异步 I/O 的实现是系统会把负责把数据从内核空间拷贝到用户空间，无需线程自己再进行阻塞的读写，内核已经准备完成。

### Select 机制

#### API 简介

linux 系统中`/usr/include/sys/select.h`文件中对`select`方法的定义如下：

```c
/* fd_set for select and pselect.  */
typedef struct
  {
    /* XPG4.2 requires this member name.  Otherwise avoid the name
       from the global namespace.  */
    #ifdef __USE_XOPEN
        __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
    # define __FDS_BITS(set) ((set)->fds_bits)
    #else
        __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
    # define __FDS_BITS(set) ((set)->__fds_bits)
    #endif
  } fd_set;

/* Check the first NFDS descriptors each in READFDS (if not NULL) for read
   readiness, in WRITEFDS (if not NULL) for write readiness, and in EXCEPTFDS
   (if not NULL) for exceptional conditions.  If TIMEOUT is not NULL, time out
   after waiting the interval specified therein.  Returns the number of ready
   descriptors, or -1 for errors.

   This function is a cancellation point and therefore not marked with
   __THROW.  */
extern int select (int __nfds, fd_set *__restrict __readfds,
                   fd_set *__restrict __writefds,
                   fd_set *__restrict __exceptfds,
                   struct timeval *__restrict __timeout);
```

- **int \_\_nfds**是`fd_set`中最大的描述符+1，当调用 select 时，内核态会判断 fd_set 中描述符是否就绪，\_\_nfds 告诉内核最多判断到哪一个描述符。

- ****readfds、**writefds、\_\_exceptfds**都是结构体`fd_set`，fd_set 可以看作是一个描述符的集合。 select 函数中存在三个 fd_set 集合，分别代表三种事件，`readfds`表示读描述符集合，`writefds`表示读描述符集合，`exceptfds`表示异常描述符集合。当对应的 fd_set = NULL 时，表示不监听该类描述符。

- **timeval \_\_timeout**用来指定 select 的工作方式，即当文件描述符尚未就绪时，select 是永远等下去，还是等待一定的时间，或者是直接返回

- **函数返回值 int**表示： 就绪描述符的数量，如果为-1 表示产生错误 。

#### 运行机制

Select 会将全量`fd_set`从用户空间拷贝到内核空间，并注册回调函数， 在内核态空间来判断每个请求是否准备好数据 。select 在没有查询到有文件描述符就绪的情况下，将一直阻塞（I/O 多路服用中提过：select 是一个阻塞函数）。如果有一个或者多个描述符就绪，那么 select 将就绪的文件描述符置位，然后 select 返回。返回后，由程序遍历查看哪个请求有数据。

#### Select 的缺陷

- 每次调用 select，都需要把 fd 集合从用户态拷贝到内核态，fd 越多开销则越大；
- 每次调用 select 都需要在内核遍历传递进来的所有 fd，这个开销在 fd 很多时也很大
- select 支持的文件描述符数量有限，默认是 1024。参见`/usr/include/linux/posix_types.h`中的定义：

`# define __FD_SETSIZE 1024`

### Poll 机制

#### API 简介

linux 系统中`/usr/include/sys/poll.h`文件中对`poll`方法的定义如下：

```c
/* Data structure describing a polling request.  */
struct pollfd
  {
    int fd;                     /* File descriptor to poll.  */
    short int events;           /* Types of events poller cares about.  */
    short int revents;          /* Types of events that actually occurred.  */
  };

/* Poll the file descriptors described by the NFDS structures starting at
   FDS.  If TIMEOUT is nonzero and not -1, allow TIMEOUT milliseconds for
   an event to occur; if TIMEOUT is -1, block until an event occurs.
   Returns the number of file descriptors with events, zero if timed out,
   or -1 for errors.

   This function is a cancellation point and therefore not marked with
   __THROW.  */
extern int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
```

- **\_\_fds**参数时 Poll 机制中定义的结构体`pollfd`，用来指定一个需要监听的描述符。结构体中 fd 为需要监听的文件描述符，events 为需要监听的事件类型，而 revents 为经过 poll 调用之后返回的事件类型，在调用 poll 的时候，一般会传入一个 pollfd 的结构体数组，数组的元素个数表示监控的描述符个数。

- **\_\_nfds**和**\_\_timeout**参数都和 Select 机制中的同名参数含义类似

#### 运行机制

poll 的实现和 select 非常相似，只是描述 fd 集合的方式不同，poll 使用`pollfd`结构代替 select 的`fd_set`（网上讲：类似于位图）结构，其他的本质上都差不多。所以**Poll 机制突破了 Select 机制中的文件描述符数量最大为 1024 的限制**。

#### Poll 的缺陷

Poll 机制相较于 Select 机制中，解决了文件描述符数量上限为 1024 的缺陷。但另外两点缺陷依然存在：

- 每次调用 poll，都需要把 fd 集合从用户态拷贝到内核态，fd 越多开销则越大；
- 每次调用 poll，都需要在内核遍历传递进来的所有 fd，这个开销在 fd 很多时也很大

### Epoll 机制

Epoll 在 Linux2.6 内核正式提出，是基于事件驱动的 I/O 方式。相对于 select 来说，epoll 没有描述符个数限制；使用一个文件描述符管理多个描述符，将用户关心的文件描述符的事件存放到内核的一个事件表中，通过内存映射，使其在用户空间也可直接访问，省去了拷贝带来的资源消耗。

#### API 简介

linux 系统中`/usr/include/sys/epoll.h`文件中有如下方法：

```c
/* Creates an epoll instance.  Returns an fd for the new instance.
   The "size" parameter is a hint specifying the number of file
   descriptors to be associated with the new instance.  The fd
   returned by epoll_create() should be closed with close().  */
extern int epoll_create (int __size) __THROW;

/* Manipulate an epoll instance "epfd". Returns 0 in case of success,
   -1 in case of error ( the "errno" variable will contain the
   specific error code ) The "op" parameter is one of the EPOLL_CTL_*
   constants defined above. The "fd" parameter is the target of the
   operation. The "event" parameter describes which events the caller
   is interested in and any associated user data.  */
extern int epoll_ctl (int __epfd, int __op, int __fd,
                      struct epoll_event *__event) __THROW;

/* Wait for events on an epoll instance "epfd". Returns the number of
   triggered events returned in "events" buffer. Or -1 in case of
   error with the "errno" variable set to the specific error code. The
   "events" parameter is a buffer that will contain triggered
   events. The "maxevents" is the maximum number of events to be
   returned ( usually size of "events" ). The "timeout" parameter
   specifies the maximum wait time in milliseconds (-1 == infinite).

   This function is a cancellation point and therefore not marked with
   __THROW.  */
extern int epoll_wait (int __epfd, struct epoll_event *__events,
                       int __maxevents, int __timeout);
```

- **epoll_create**函数：创建一个 epoll 实例并返回，该实例可以用于监控\_\_size 个文件描述符

- **epoll_ctl**函数：向 epoll 中注册事件，该函数如果调用成功返回 0，否则返回-1。

  - \_\_epfd 为 epoll_create 返回的 epoll 实例
  - \_\_op 表示要进行的操作
  - \_\_fd 为要进行监控的文件描述符
  - \_\_event 要监控的事件

- **epoll_wait**函数：类似与 select 机制中的 select 函数、poll 机制中的 poll 函数，等待内核返回监听描述符的事件产生。该函数返回已经就绪的事件的数量，如果为-1 表示出错。
  - \_\_epfd 为 epoll_create 返回的 epoll 实例
  - \_\_events 数组为 epoll_wait 要返回的已经产生的事件集合
  - **maxevents 为希望返回的最大的事件数量（通常为**events 的大小）
  - \_\_timeout 和 select、poll 机制中的同名参数含义相同

#### 运行机制

epoll 操作过程需要上述三个函数，也正是通过三个函数完成 Select 机制中一个函数完成的事情，解决了 Select 机制的三大缺陷。epoll 的工作机制更为复杂，我们就解释一下，它是如何解决 Select 机制的三大缺陷的。

1. 对于第一个缺点，epoll 的解决方案是：它的**fd 是共享在用户态和内核态之间**的，所以可以不必进行从用户态到内核态的一个拷贝，大大节约系统资源。至于如何做到用户态和内核态，大家可以查一下“**mmap**”，它是一种内存映射的方法。
2. 对于第二个缺点，epoll 的解决方案不像 select 或 poll 一样每次都把当前线程轮流加入 fd 对应的设备等待队列中，而只在 epoll_ctl 时把当前线程挂一遍（这一遍必不可少），并为每个 fd 指定一个回调函数。当设备就绪，唤醒等待队列上的等待者时，就会调用这个回调函数，而**这个回调函数会把就绪的 fd 加入一个就绪链表。那么当我们调用 epoll_wait 时，epoll_wait 只需要检查链表中是否有存在就绪的 fd 即可，效率非常可观**。
3. 对于第三个缺点，fd 数量的限制，也只有 Select 存在，Poll 和 Epoll 都不存在。由于 Epoll 机制中只关心就绪的 fd，它相较于 Poll 需要关心所有 fd，在连接较多的场景下，效率更高。在 1GB 内存的机器上大约是 10 万左右，一般来说这个数目和系统内存关系很大。

#### 工作模式

相较于 Select 和 Poll，Epoll 内部还分为两种工作模式： **LT 水平触发（level trigger）** 和 **ET 边缘触发（edge trigger）**。

- **LT 模式：**  默认的工作模式，即当 epoll_wait 检测到某描述符事件就绪并通知应用程序时，应用程序**可以不立即处理**该事件；事件会被放回到就绪链表中，下次调用 epoll_wait 时，会再次通知此事件。
- **ET 模式：**  当 epoll_wait 检测到某描述符事件就绪并通知应用程序时，应用程序**必须立即处理**该事件。如果不处理，下次调用 epoll_wait 时，不会再次响应并通知此事件。

由于上述两种工作模式的区别，LT 模式同时支持 block 和 no-block socket 两种，而 ET 模式下仅支持 no-block socket。即 epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个 fd 的阻塞 I/O 操作把多个处理其他文件描述符的任务饿死。ET 模式在很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。

#### Epoll 的优点

- 使用内存映射技术，节省了用户态和内核态间数据拷贝的资源消耗；
- 通过每个 fd 定义的回调函数来实现的，只有就绪的 fd 才会执行回调函数。I/O 的效率不会随着监视 fd 的数量的增长而下降；
- 文件描述符数量不再受限；

### Select、Poll、Epoll 机制的对比

下图主流 I/O 多路复用机制的 benchmark：

![](https://pic4.zhimg.com/80/v2-b89e1b4d3869eff5ed1cfe3aec2512eb_720w.webp)

当并发 fd 较小时，Select、Poll、Epoll 的响应效率想差无几，甚至 Select 和 Poll 更胜一筹。但是当并发连接（fd）较多时，Epoll 的优势便真正展现出来。

下面一张表格总结三种模式的区别：

![](https://pic3.zhimg.com/80/v2-b23f9ccc4a82c26cc447779aa4427a0e_720w.webp)

通过上述的一些总结，希望我们对 I/O 多路复用的 Select、Poll、Epoll 机制有一个更深刻的认识。也要明白为什么 epoll 会成为 Linux 平台下实现高性能网络服务器的首选 I/O 多路复用机制。

### Epoll 的使用场景

上面的文章中已经不断介绍了 Epoll 机制的优势，又提到它是**Linux 平台下实现高性能网络服务器的首选 I/O 复用机制。实际工作中，我们在哪里会用到它？怎么用呢？**

比如下面代码，就是我们使用高性能网络框架 Netty 实现 IM 项目中对于 netty 的 bossGroup 和 workerGroup 以及 serverChannel 的配置

```java
String os = System.getProperty("os.name");
if(os.toLowerCase().startsWith("win") || os.toLowerCase().startsWith("mac")){
    // 点开NioEventLoopGroup的源码，对于这个类是这么注释的
    // MultithreadEventLoopGroup implementations which is used for NIO Selector based Channel
    bossGroup = new NioEventLoopGroup(1);
    workerGroup = new NioEventLoopGroup(4);
}else{
    // 点开EpollEventLoopGroup的源码，对于这个类是这么注释的
    // EventLoopGroup which uses epoll under the covers. Because of this it only works on linux.
    bossGroup = new EpollEventLoopGroup(1);
    workerGroup = new EpollEventLoopGroup(4);
}
bootStrap = new ServerBootstrap();
bootStrap.group(bossGroup,workerGroup);
if(os.toLowerCase().startsWith("win") || os.toLowerCase().startsWith("mac")) {
    // NioServerSocketChannel implementation which uses NIO selector based implementation to accept new connections.
    bootStrap.channel(NioServerSocketChannel.class);
}else{
    // ServerSocketChannel implementation that uses linux EPOLL Edge-Triggered Mode for maximal performance.
    // 注意看注释中的“linux EPOLL Edge-Triggered Mode”，linux下ET模式的Epoll机制
    bootStrap.channel(EpollServerSocketChannel.class);
}
```
