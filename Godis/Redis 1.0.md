---
date created: 2023-03-08 14:10
---

#缓存中间件 #epoll

## 如何深入理解新事物

核心概念/定义 => 核心流程(先忽略错误处理等) => 核心数据结构 (功能实现 -> 优化细节)

核心流程:

- ![[Pasted image 20230119174553.png]]

- 客户端请求服务端, 服务端打开事件循环, 事件循环内监听网络请求, 监听到后分配内存资源
- redis.c/redis-cli.c -> ae.c -> anet.c -> zmalloc.c

核心数据结构:

- 字符串 + 通用双向链表 + 通用字典
- sds.c + adlist.c + dict.c

工具:

- pqsort.c + benchmark.c + lzf.c

考虑到redis是单线程 + 云原生工具, 一般用小机器(1核2G)部署单实例 + 多集群部署, 这样机器利用率高, 但也带来高运维成本

## 集群方式

- smart client: 底层实现为hash映射, 数据迁移成本高, 只能用作功能验证
- redis cluster: 用gossip协议保证一致性, 只适用于集群<500的小集群, 集群多了为了维护一致性对带宽压力大
- proxy: 在集群上加proxy层, 相当于二次哈希, 适用于超大集群

## 运维问题

- 单核机器运行多redis实例?
  - Dragonfly: 把CPU做映射, 切分为不同的块

## 持久化问题

- 自带DBA & AOF太菜
  - 添加南向超大缓存层, 先把超大缓存当磁盘用
  - 基于levelDB的组合落盘思想, 消费超大缓存到磁盘中

## IO多路复用

### 经典网络IO过程

![[Pasted image 20230119175504.png]]

1. 客户端client将包packet发送到服务器的网卡NIC
2. 对于现代网卡, 直接使用DMA技术将packet内容拷贝到驱动管理的内核态内存RingBuffer
   对于老旧网卡, 没有DMA, 使用IRQ硬中断CPU, 让CPU完成拷贝
3. 拷贝完成后, 网卡驱动使用IRQ硬中断将权限交予CPU, CPU为了减少阻塞事件, 立即屏蔽硬中断. 并调用`ksoftirqd`软中断指令, 执行驱动提供的`poll`函数, 将RingBuffer内容读到skb(socket buffer)
4. CPU继续执行内核函数`net_if_receive_skb`, 将驱动里的skb通过对应网络层协议栈拆包, 结果写入socket接收队列
5. 一旦队列有数据, 改变状态为readable, 应用程序代码便可以将队列内容拷贝到用户内存, 执行对应逻辑
6. 一旦RingBuffer读空, 则取消中断屏蔽

### IO多路复用

> socket队列有很多, 对应不同的应用注册的socket, 应用程序调用read命令时并不知道哪些socket是有数据的. 如果是阻塞式IO, 则当socket没有数据时, 应用线程就阻塞住, 这对于单线程的Redis是不可接受的.

![[Pasted image 20230119183620.png]]

- 在经典模型的第5步, 即数据通过软中断写入socket队列, 状态变为readable后, 应用程序取数据时
  - 如果是`select`调用, 则轮询所有注册的socket, 将readable的socket的fd形成**数组就绪队列**`fds[]`, 这样应用程序在fds里挑出的socket执行对应操作一定不会被阻塞
  - 如果是poll调用, 则是**链表就绪队列**`*fds`
  - 对于支持`epoll`的机器, 则是`epoll_ctl`使用软中断将readable的socket包装成`epoll_item`并放入**红黑树就绪队列**`*rbfds`, `epoll_wait`用于从红黑树中找到所需`epoll_item`对应的socket的fd
