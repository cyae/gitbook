![[Pasted image 20230312173039.png]]

## Socket 的基本操作函数

### socket()函数

```c
int socket(int domain, int type, int protocol);
```

socket 函数对应于普通文件的打开操作。普通文件的打开操作返回一个文件描述字，而 socket()用于创建一个 socket 描述符（socket descriptor），它唯一标识一个 socket。这个 socket 描述字跟文件描述字一样，后续的操作都有用到它，把它作为参数，通过它来进行一些读写操作。

正如可以给 fopen 的传入不同参数值，以打开不同的文件。创建 socket 的时候，也可以指定不同的参数创建不同的 socket 描述符，socket 函数的三个参数分别为：

- domain：即协议域，又称为协议族（family）。常用的协议族有，AF_INET（IPv4)、AF_INET6(IPv6)、AF_LOCAL（或称 AF_UNIX，Unix 域 socket）、AF_ROUTE 等等。协议族决定了 socket 的地址类型，在通信中必须采用对应的地址，如 AF_INET 决定了要用 ipv4 地址（32 位的）与端口号（16 位的）的组合、AF_UNIX 决定了要用一个绝对路径名作为地址。
- type：指定 socket 类型。常用的 socket 类型有，SOCK_STREAM（流式套接字）、SOCK_DGRAM（数据报式套接字）、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET 等等
- protocol：就是指定协议。常用的协议有，IPPROTO_TCP、PPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC 等，它们分别对应 TCP 传输协议、UDP 传输协议、STCP 传输协议、TIPC 传输协议。

**返回值**：

若无错误发生，socket()返回引用新套接口的描述字。否则的话，返回 INVALID_SOCKET 错误，应用程序可通过 WSAGetLastError()获取相应错误代码。

**注意**：

并不是上面的 type 和 protocol 可以随意组合的，如 SOCK_STREAM 不可以跟 IPPROTO_UDP 组合。当 protocol 为 0 时，会自动选择 type 类型对应的默认协议。

#### 命名 socket

SOCK_STREAM 式套接字的通信双方均需要具有地址，其中服务器端的地址需要明确指定，ipv4 的指定方法是使用 struct sockaddr_in 类型的变量。

```c
struct sockaddr_in {
    sa_family_t    sin_family; /* address family: AF_INET */
    in_port_t      sin_port;   /* port in network byte order */
    struct in_addr sin_addr;   /* internet address */
};
/* Internet address. */
struct in_addr {
    uint32_t       s_addr;     /* address in network byte order */
};
struct sockaddr_in     servaddr;
memset(&servaddr, 0, sizeof(servaddr));
servaddr.sin_family = AF_INET;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//IP地址设置成INADDR_ANY,让系统自动获取本机的IP地址。
servaddr.sin_port = htons(DEFAULT_PORT);//设置的端口
```

INADDR_ANY 就是指定地址为 0.0.0.0 的地址，这个地址事实上表示不确定地址，或“所有地址”、“任意地址”。也就是表示本机的所有 IP，因为有些机子不止一块网卡，多网卡的情况下，这个就表示所有网卡 ip 地址的意思。
客户端 connect 时，不能使用 INADDR_ANY 选项。必须指明要连接哪个服务器 IP。

- htons 将主机的无符号短整形数转换成网络字节顺序
- htonl 将主机的无符号长整形数转换成网络字节顺序

#### 网络字节序与主机字节序

主机字节序就是我们平常说的大端和小端模式：不同的 CPU 有不同的字节序类型，这些字节序是指整数在内存中保存的顺序，这个叫做主机序。引用标准的 Big-Endian 和 Little-Endian 的定义如下：

- Little-Endian 就是低位字节排放在内存的低地址端，高位字节排放在内存的高地址端。
- Big-Endian 就是高位字节排放在内存的低地址端，低位字节排放在内存的高地址端。

网络字节序：4 个字节的 32 bit 值以下面的次序传输：首先是 0 ～ 7bit，其次 8 ～ 15bit，然后 16 ～ 23bit，最后是 24~31bit。这种传输次序称作大端字节序。由于 TCP/IP 首部中所有的二进制整数在网络中传输时都要求以这种次序，因此它又称作网络字节序。字节序，顾名思义字节的顺序，就是大于一个字节类型的数据在内存中的存放顺序，一个字节的数据没有顺序的问题了。

在将一个地址绑定到 socket 的时候，请先将主机字节序转换成为网络字节序，而不要假定主机字节序跟网络字节序一样使用的是 Big-Endian。谨记对主机字节序不要做任何假定，务必将其转化为网络字节序再赋给 socket。

当我们调用 socket 创建一个 socket 时，返回的 socket 描述字它存在于协议族（address family，AF_XXX）空间中，但没有一个具体的地址。如果想要给它赋值一个地址，就必须调用 bind()函数，否则就当调用 connect()、listen()时系统会自动随机分配一个端口。

### bind()函数

bind()函数把一个地址族中的特定地址赋给 socket。例如对应 AF_INET、AF_INET6 就是把一个 ipv4 或 ipv6 地址和端口号组合赋给 socket。

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

函数的三个参数分别为：

- sockfd：即 socket 描述字，它是通过 socket()函数创建了，唯一标识一个 socket。bind()函数就是将给这个描述字绑定一个名字。
- addrlen：对应的是地址的长度。
  - 通常服务器在启动的时候都会绑定一个众所周知的地址（如 ip 地址+端口号），用于提供服务，客户就可以通过它来接连服务器；而客户端就不用指定，有系统自动分配一个端口号和自身的 ip 地址组合。这就是为什么通常服务器端在 listen 之前会调用 bind()，而客户端就不会调用，而是在 connect()时由系统随机生成一个。
- addr：一个 const struct sockaddr \*指针，指向要绑定给 sockfd 的协议地址。

```c
struct sockaddr{
    sa_family_t  sin_family;   //地址族（Address Family），也就是地址类型
    char         sa_data[14];  //IP地址和端口号
};
```

sockaddr 是一种通用的结构体，可以用来保存多种类型的 IP 地址和端口号。要想给 sa_data 赋值，必须同时指明 IP 地址和端口号，例如”127.0.0.1:80“，但没有相关函数将这个字符串转换成需要的形式，也就很难给 sockaddr 类型的变量赋值。正是由于通用结构体 sockaddr 使用不便，才针对不同的地址类型定义了不同的结构体。 如 ipv6 对应的是：

```c
struct sockaddr_in6 {
    sa_family_t     sin6_family;   /* AF_INET6 */
    in_port_t       sin6_port;     /* port number */
    uint32_t        sin6_flowinfo; /* IPv6 flow information */
    struct in6_addr sin6_addr;     /* IPv6 address */
    uint32_t        sin6_scope_id; /* Scope ID (new in 2.4) */
};

struct in6_addr {
    unsigned char   s6_addr[16];   /* IPv6 address */
};
```

Unix 域对应的是：

```c
#define UNIX_PATH_MAX    108

struct sockaddr_un {
    sa_family_t sun_family;               /* AF_UNIX */
    char        sun_path[UNIX_PATH_MAX];  /* pathname */
};
```

**返回值**：
如无错误发生，则 bind()返回 0。否则的话，将返回-1，应用程序可通过 WSAGetLastError()获取相应错误代码。

### listen()、connect()函数
如果作为一个服务器，在调用 socket()、bind()之后就会调用 listen()来监听这个 socket，如果客户端这时调用 connect()发出连接请求，服务器端就会接收到这个请求。

```c
int listen(int sockfd, int backlog);
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

listen 函数的

- 第一个参数即为要监听的 socket 描述字，
- 第二个参数为相应 socket 可以排队的最大连接个数。
  socket()函数创建的 socket 默认是一个主动类型的，listen 函数将 socket 变为被动类型的，等待客户的连接请求。

**返回值**：
如无错误发生，listen()返回 0。否则的话，返回-1，应用程序可通过 WSAGetLastError()获取相应错误代码。

connect 函数的

- 第一个参数即为客户端的 socket 描述字，
- 第二参数为服务器的 socket 地址，
- 第三个参数为 socket 地址的长度。

客户端通过调用 connect 函数来建立与 TCP 服务器的连接。

**返回值**：
若无错误发生，则 connect()返回 0。否则的话，返回 SOCKET_ERROR 错误，应用程序可通过 WSAGetLastError()获取相应错误代码。对非阻塞套接口而言，若返回值为 SOCKET_ERROR 则应用程序调用 WSAGetLastError()。如果它指出错误代码为 WSAEWOULDBLOCK，则您的应用程序可以：

- 用 select()，通过检查套接口是否可写，来确定连接请求是否完成。
- 如果您的应用程序使用基于消息的 WSAAsyncSelect()来表示对连接事件的兴趣，则当连接操作完成后，您会收到一个 FD_CONNECT 消息。

### accept()函数

TCP 服务器端依次调用 socket()、bind()、listen()之后，就会监听指定的 socket 地址了。TCP 客户端依次调用 socket()、connect()之后就向 TCP 服务器发送了一个连接请求。TCP 服务器监听到这个请求之后，就会调用 accept()函数取接收请求，这样连接就建立好了。之后就可以开始网络 I/O 操作了，即类同于普通文件的读写 I/O 操作。

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

accept 函数的

- 第一个参数为服务器的 socket 描述字，
- 第二个参数为指向 struct sockaddr \*的指针，用于返回客户端的协议地址，
- 第三个参数为客户端协议地址的长度。如果 accpet 成功，那么其返回值是由内核自动生成的一个全新的描述字，代表与返回客户的 TCP 连接。

**返回值**：
如果没有错误产生，则 accept()返回一个描述所接受包的 SOCKET 类型的值。否则的话，返回 INVALID_SOCKET 错误，应用程序可通过调用 WSAGetLastError()来获得特定的错误代码。

**注意**：
accept 的第一个参数为服务器的 socket 描述字，是服务器开始调用 socket()函数生成的，称为监听 socket 描述字；
而 accept 函数返回的是已连接的 socket 描述字。两个套接字不一样。
一个服务器通常通常仅仅只创建一个监听 socket 描述字，它在该服务器的生命周期内一直存在。内核为每个由服务器进程接受的客户连接创建了一个已连接 socket 描述字，当服务器完成了对某个客户的服务，相应的已连接 socket 描述字就被关闭。

### recv()、send()等函数

至此服务器与客户已经建立好连接了。可以调用网络 I/O 进行读写操作了，即实现了网咯中不同进程之间的通信！网络 I/O 操作有下面几组：

- read()/write()
- recv()/send()
- readv()/writev()
- recvmsg()/sendmsg()
- recvfrom()/sendto()

它们的声明如下：

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
#include <sys/types.h>
#include <sys/socket.h>
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t sendto(int sockfd, const void *buf, size_t len, int  flags,const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```

read 函数是负责从 fd 中读取内容。当读成功时，read 返回实际所读的字节数，如果返回的值是 0 表示已经读到文件的结束了，小于 0 表示出现了错误。如果错误为 EINTR 说明读是由中断引起的，如果是 ECONNREST 表示网络连接出了问题。

write 函数将 buf 中的 nbytes 字节内容写入文件描述符 fd。成功时返回写的字节数。失败时返回-1，并设置 errno 变量。 在网络程序中，当我们向套接字文件描述符写时有两种可能。

- write 的返回值大于 0，表示写了部分或者是全部的数据
- 返回的值小于 0，此时出现了错误。我们要根据错误类型来处理。如果错误为 EINTR 表示在写的时候出现了中断错误。如果为 EPIPE 表示网络连接出现了问题(对方已经关闭了连接)。

recv 函数和 send 函数提供了 read 和 write 函数一样的功能，不同的是他们提供了四个参数。前面的三个参数和 read、write 函数是一样的。

**参数**

- 第一个参数指定发送端套接字描述符；
- 第二个参数指明一个存放应用程序要发送数据的缓冲区；
- 第三个参数指明实际要发送的数据的字节数；
- 第四个参数一般置 0。或者是以下组合：
  - MSG_DONTROUTE：不查找表，是 send 函数使用的标志，这个标志告诉 IP，目的主机在本地网络上，没有必要查找表，这个标志一般用在网络诊断和路由程序里面。
  - MSG_OOB：表示可以接收和发送带外数据。
  - MSG_PEEK：查看数据，并不从系统缓冲区移走数据。是 recv 函数使用的标志，表示只是从系统缓冲区中读取内容，而不清楚系统缓冲区的内容。这样在下次读取的时候，依然是一样的内容，一般在有个进程读写数据的时候使用这个标志。
  - MSG_WAITALL：等待所有数据，是 recv 函数的使用标志，表示等到所有的信息到达时才返回，使用这个标志的时候，recv 返回一直阻塞，直到指定的条件满足时，或者是发生了错误。

#### 同步 Socket 的 send 函数的执行流程

当调用该函数时，

1. send 先比较待发送数据的长度 len 和套接字 s 的发送缓冲的长度， 如果 len 大于 s 的发送缓冲区的长度，该函数返回 SOCKET_ERROR；
2. 如果 len 小于或者等于 s 的发送缓冲区的长度，那么 send 先检查协议 s 的发送缓冲中的数据是否正在发送，如果是就等待协议把数据发送完，如果协议还没有开始发送 s 的发送缓冲中的数据或者 s 的发送缓冲中没有数据，那么 send 就比较 s 的发送缓冲区的剩余空间和 len
3. 如果 len 大于剩余空间大小，send 就一直等待协议把 s 的发送缓冲中的数据发送完
4. 如果 len 小于剩余 空间大小，send 就仅仅把 buf 中的数据 copy 到剩余空间里（注意并不是 send 把 s 的发送缓冲中的数据传到连接的另一端的，而是协议传的，send 仅仅是把 buf 中的数据 copy 到 s 的发送缓冲区的剩余空间里）。

如果 send 函数 copy 数据成功，就返回实际 copy 的字节数，如果 send 在 copy 数据时出现错误，那么 send 就返回 SOCKET_ERROR；如果 send 在等待协议传送数据时网络断开的话，那么 send 函数也返回 SOCKET_ERROR。

**注意**：

- send 函数把 buf 中的数据成功 copy 到 s 的发送缓冲的剩余空间里后它就返回了，但是此时这些数据并不一定马上被传到连接的另一端。如果协议在后续的传送过程中出现网络错误的话，那么下一个 socket 函数就会返回 SOCKET_ERROR。（每一个除 send 外的 socket 函数在执 行的最开始总要先等待套接字的发送缓冲中的数据被协议传送完毕才能继续，如果在等待时出现网络错误，那么该 Socket 函数就返回 SOCKET_ERROR）
- 在 Unix 系统下，如果 send 在等待协议传送数据时网络断开的话，调用 send 的进程会接收到一个 SIGPIPE 信号，进程对该信号的默认处理是进程终止。

通过测试发现，异步 socket 的 send 函数在网络刚刚断开时还能发送返回相应的字节数，同时使用 select 检测也是可写的，但是过几秒钟之后，再 send 就会出错了，返回-1。select 也不能检测出可写了。

#### 同步 Socket 的 recv 函数的执行流程

当应用程序调用 recv 函数时，

1. recv 先等待 s 的发送缓冲中的数据被协议传送完毕，如果协议在传送 s 的发送缓冲中的数据时出现网络错误，那么 recv 函数返回 SOCKET_ERROR，

2. 如果 s 的发送缓冲中没有数据或者数据被协议成功发送完毕后，recv 先检查套接字 s 的接收缓冲区，如果 s 接收缓冲区中没有数据或者协议正在接收数据，那么 recv 就一直等待，直到协议把数据接收完毕。当协议把数据接收完毕，recv 函数就把 s 的接收缓冲中的数据 copy 到 buf 中（注意协议接收到的数据可能大于 buf 的长度，所以 在这种情况下要调用几次 recv 函数才能把 s 的接收缓冲中的数据 copy 完。recv 函数仅仅是 copy 数据，真正的接收数据是协议来完成的），recv 函数返回其实际 copy 的字节数。如果 recv 在 copy 时出错，那么它返回 SOCKET_ERROR；如果 recv 函数在等待协议接收数据时网络中断了，那么它返回 0。

**注意**：
在 Unix 系统下，如果 recv 函数在等待协议接收数据时网络断开了，那么调用 recv 的进程会接收到一个 SIGPIPE 信号，进程对该信号的默认处理是进程终止。

### select()函数

connect、accept、recv 或 recvfrom 这样的阻塞程序（所谓阻塞方式 block，顾名思义，就是进程或是线程执行到这些函数时必须等待某个事件的发生，如果事件没有发生，进程或线程就被阻塞，函数不能立即返回）。

可是使用 Select 就可以完成非阻塞（所谓非阻塞方式 non-block，就是进程或线程执行此函数时不必非要等待事件的发生，一旦执行肯定返回，以返回值的不同来反映函数的执行情况，如果事件发生则与阻塞方式相同，若事件没有发生则返回一个代码来告知事件未发生，而进程或线程继续执行，所以效率较高）方式工作的程序，它能够监视我们需要监视的文件描述符的变化情况——读写或是异常。

```c
int select(int maxfdp,fd_set *readfds,fd_set *writefds,fd_set *errorfds,struct timeval*timeout);
```

- struct fd_set 可以理解为一个集合，这个集合中存放的是文件描述符(filedescriptor)，即文件句柄，fd_set 集合可以通过一些宏由人为来操作。

```c
FD_ZERO(fd_set *set);       //Clear all entries from the set.
FD_SET(int fd, fd_set *set);    //Add fd to the set.
FD_CLR(int fd, fd_set *set);    //Remove fd from the set.
FD_ISSET(int fd, fd_set *set);  //Return true if fd is in the set.
```

- struct timeval 代表时间值。

```c
struct timeval {
int tv_sec;     //seconds
int tv_usec;    //microseconds，注意这里是微秒不是毫秒
};
```

**参数**：

- int maxfdp 是一个整数值，是指集合中所有文件描述符的范围，即所有文件描述符的最大值加 1。
- fd_set \* readfds 是指向 fd_set 结构的指针，这个集合中应该包括文件描述符，我们是要监视这些文件描述符的读变化的，即我们关心是否可以从这些文件中读取数据了，如果这个集合中有一个文件可读，select 就会返回一个大于 0 的值，表示有文件可读，如果没有可读的文件，则根据 timeout 参数再判断是否超时，若超出 timeout 的时间，select 返回 0，若发生错误返回负值。可以传入 NULL 值，表示不关心任何文件的读变化。
- fd_set \* writefds 是指向 fd_set 结构的指针，这个集合中应该包括文件描述符，我们是要监视这些文件描述符的写变化的，即我们关心是否可以向这些文件中写入数据了，如果这个集合中有一个文件可写，select 就会返回一个大于 0 的值，表示有文件可写，如果没有可写的文件，则根据 timeout 参数再判断是否超时，若超出 timeout 的时间，select 返回 0，若发生错误返回负值。可以传入 NULL 值，表示不关心任何文件的写变化。
- fd_set \* errorfds 同上面两个参数的意图，用来监视文件错误异常。
- struct timeval \* timeout 是 select 的超时时间，这个参数至关重要，它可以使 select 处于三种状态，第一，若将 NULL 以形参传入，即不传入时间结构，就是将 select 置于阻塞状态，一定等到监视文件描述符集合中某个文件描述符发生变化为止；第二，若将时间值设为 0 秒 0 毫秒，就变成一个纯粹的非阻塞函数，不管文件描述符是否有变化，都立刻返回继续执行，文件无变化返回 0，有变化返回一个正值；第三，timeout 的值大于 0，这就是等待的超时时间，即 select 在 timeout 时间内阻塞，超时时间之内有事件到来就返回了，否则在超时后不管怎样一定返回，返回值同上述。

**返回值**：
返回状态发生变化的描述符总数。 负值：select 错误 ；正值：某些文件可读写或出错 ；0：等待超时，没有可读写或错误的文件

#### 理解 select 模型：

理解 select 模型的关键在于理解 fd_set,为说明方便，取 fd_set 长度为 1 字节，fd_set 中的每一 bit 可以对应一个文件描述符 fd。则 1 字节长的 fd_set 最大可以对应 8 个 fd。

1. 执行 fd_set set;FD_ZERO(&set);则 set 用位表示是 0000,0000。
2. 若 fd ＝ 5,执行 FD_SET(fd,&set);后 set 变为 0001,0000(第 5 位置为 1)
3. 若再加入 fd ＝ 2，fd=1,则 set 变为 0001,0011
4. 执行 select(6,&set,0,NULL,NULL)阻塞等待
5. 若 fd=1,fd=2 上都发生可读事件，则 select 返回，此时 set 变为 0000,0011。注意：没有事件发生的 fd=5 被清空。

基于上面的讨论，可以轻松得出 select 模型的特点：

#### select 模型的特点

- 可监控的文件描述符个数取决与 sizeof(fd_set)的值。我这边服务器上 sizeof(fd_set)＝ 512，每 bit 表示一个文件描述符，则我服务器上支持的最大文件描述符是 512\*8=4096。据说可调，另有说虽然可调，但调整上限受于编译内核时的变量值。

- 将 fd 加入 select 监控集的同时，还要再使用一个数据结构 array 保存放到 select 监控集中的 fd，一是用于再 select 返回后，array 作为源数据和 fd_set 进行 FD_ISSET 判断。二是 select 返回后会把以前加入的但并无事件发生的 fd 清空，则每次开始 select 前都要重新从 array 取得 fd 逐一加入（FD_ZERO 最先），扫描 array 的同时取得 fd 最大值 maxfd，用于 select 的第一个参数。

- 可见 select 模型必须在 select 前循环 array（加 fd，取 maxfd），select 返回后循环 array（FD_ISSET 判断是否有时间发生）。
  使用 select 和 non-blocking 实现 server 处理多 client 实例
  SELECT

![[Pasted image 20230312173136.jpg]]

### close()/shutdown()函数

```c
int close(int sockfd);
```

close 一个套接字的默认行为是把套接字标记为已关闭，然后立即返回到调用进程，该套接字描述符不能再由调用进程使用，也就是说它不能再作为 read 或 write 的第一个参数，然而 TCP 将尝试发送已排队等待发送到对端，发送完毕后发生的是正常的 TCP 连接终止序列。

#### 多进程中 close 操作解释

在多进程并发服务器中，父子进程共享着套接字，套接字描述符引用计数记录着共享着的进程个数，当父进程或某一子进程 close 掉套接字时，描述符引用计数会相应的减一，当引用计数仍大于零时，这个 close 调用就不会引发 TCP 的四路握手断连过程。

```c
int shutdown(int sockfd,int howto);
```

该函数的行为依赖于 howto 的值

- SHUT_RD：值为 0，关闭连接的读这一半。
- SHUT_WR：值为 1，关闭连接的写这一半。
- SHUT_RDWR：值为 2，连接的读和写都关闭。
  终止网络连接的通用方法是调用 close 函数。但使用 shutdown 能更好的控制断连过程（使用第二个参数）。

#### close 与 shutdown 的区别

主要表现在：

- close 函数会关闭套接字 ID，如果有其他的进程共享着这个套接字，那么它仍然是打开的，这个连接仍然可以用来读和写，并且有时候这是非常重要的 ，特别是对于多进程并发服务器来说。
- 而 shutdown 会切断进程共享的套接字的所有连接，不管这个套接字的引用计数是否为零，那些试图读得进程将会接收到 EOF 标识，那些试图写的进程将会检测到 SIGPIPE 信号，同时可利用 shutdown 的第二个参数选择断连的方式。
