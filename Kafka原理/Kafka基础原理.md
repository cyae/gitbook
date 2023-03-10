---
date created: 2023-03-08 14:05
date updated: 2023-03-08 14:06
---

#消息中间件

> [source](https://www.modb.pro/db/132102)

## 一、Kafka 基础

引入一个场景，我们知道中国移动，中国联通，中国电信的日志处理，是交给外包去做大数据分析的，假设现在它们的日志都交给了你做的系统去做用户画像分析。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_85a11006-2b45-11ec-b625-fa163eb4f6be.png)

按照刚刚前面提到的消息系统的作用，我们知道了消息系统其实就是一个**模拟缓存**，且**仅仅是起到了缓存的作用**而并不是真正的缓存，数据仍然是存储在磁盘上面而不是内存。

### 1.Topic 主题

kafka 学习了数据库里面的设计，在里面设计了 topic（主题），这个东西类似于关系型数据库的表

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_85bf2fbe-2b45-11ec-b625-fa163eb4f6be.png)

此时我需要获取中国移动的数据，那就直接监听 TopicA 即可

### 2.Partition 分区

kafka 还有一个概念叫 Partition（分区），分区具体在服务器上面表现起初就是一个目录，一个主题下面有多个分区，这些分区会存储到不同的服务器上面，或者说，其实就是在不同的主机上建了不同的目录。这些分区主要的信息就存在了.log 文件里面。跟数据库里面的分区差不多，是为了提高性能。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_85f46f4e-2b45-11ec-b625-fa163eb4f6be.png)

至于为什么提高了性能，很简单，多个分区多个线程，多个线程并行处理肯定会比单线程好得多

Topic 和 partition 像是 HBASE 里的 table 和 region 的概念，table 只是一个逻辑上的概念，真正存储数据的是 region，这些 region 会分布式地存储在各个服务器上面，对应于 kafka，也是一样，**Topic 也是逻辑概念**，而 partition 就是分布式存储单元。这个设计是保证了海量数据处理的基础。我们可以对比一下，如果 HDFS 没有 block 的设计，一个 100T 的文件也只能单独放在一个服务器上面，那就直接占满整个服务器了，引入 block 后，大文件可以分散存储在不同的服务器上。

注意：

- 分区会有单点故障问题，所以我们会为每个分区设置副本数
- 分区的编号是从 0 开始的

### 3.Producer - 生产者

往消息系统里面发送数据的就是生产者

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_8624d170-2b45-11ec-b625-fa163eb4f6be.png)

### 4.Consumer - 消费者

从 kafka 里读取数据的就是消费者

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_865ea10c-2b45-11ec-b625-fa163eb4f6be.png)

### 5.Message - 消息

kafka 里面的我们处理的数据叫做消息

## 二、kafka 的集群架构

创建一个 TopicA 的主题，3 个分区分别存储在不同的服务器，也就是 broker 下面。**Topic 是一个逻辑上的概念**，并不能直接在图中把 Topic 的相关单元画出

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_869d4056-2b45-11ec-b625-fa163eb4f6be.png)

**需要注意：kafka 在 0.8 版本以前是没有副本机制的，所以在面对服务器宕机的突发情况时会丢失数据，所以尽量避免使用这个版本之前的 kafka**

### 1.Replica - 副本

kafka 中的 partition 为了保证数据安全，所以每个 partition 可以设置多个副本。

此时我们对分区 0,1,2 分别设置 3 个副本（其实设置两个副本是比较合适的）

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_86c787ee-2b45-11ec-b625-fa163eb4f6be.png)

而且其实每个副本都是有角色之分的，它们会选取一个副本作为 leader，而其余的作为 follower，我们的**生产者在发送数据的时候，是直接发送到 leader partition 里面**，然后 follower partition 会去 leader 那里自行同步数据，**消费者消费数据的时候，也是从 leader 那去消费数据的**。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_86ee7fc0-2b45-11ec-b625-fa163eb4f6be.png)

### 2.Consumer Group - 消费者组

我们在消费数据时会在代码里面指定一个 group.id,这个 id 代表的是消费组的名字，而且**这个 group.id 就算不设置，系统也会默认设置**

```java
conf.setProperty("group.id","tellYourDream")
```

我们所熟知的一些消息系统一般来说会这样设计，就是只要有一个消费者去消费了消息系统里面的数据，那么其余所有的消费者都不能再去消费这个数据。可是 kafka 并不是这样,比如现在 consumerA 去消费了一个 topicA 里面的数据。

```yaml
consumerA:
group.id = a
consumerB:
group.id = a
consumerC:
group.id = b
consumerD:
group.id = b
```

再让 consumerB 也去消费 TopicA 的数据，它是消费不到了，但是我们在 consumerC 中重新指定一个另外的 group.id，consumerC 是可以消费到 topicA 的数据的。而 consumerD 也是消费不到的，所以在 kafka 中，**不同组可有唯一的一个消费者去消费同一主题的数据**。

所以消费者组就是让多个消费者并行消费信息而存在的，而且它们不会消费到同一个消息，如下，consumerA，B，C 是不会互相干扰的

```yaml
consumer group:a
consumerA
consumerB
consumerC
```

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_8733b306-2b45-11ec-b625-fa163eb4f6be.png)

如图，因为前面提到过了消费者会直接和 leader 建立联系，所以它们分别消费了三个 leader，所以**一个分区不会让消费者组里面的多个消费者去消费**，但是在消费者不饱和的情况下，**一个消费者是可以去消费多个分区的数据的**。

### 3.Controller

熟知一个规律：在大数据分布式文件系统里面，95%的都是主从式的架构，个别是对等式的架构，比如 ElasticSearch。

kafka 也是主从式的架构，主节点就叫 controller，其余的为从节点，controller 是需要和 zookeeper 进行配合管理整个 kafka 集群。

### 4.kafka 和 zookeeper 如何配合工作

kafka 严重依赖于 zookeeper 集群（所以之前的 zookeeper 文章还是有点用的）。所有的 broker 在启动的时候都会往 zookeeper 进行注册，目的就是选举出一个 controller，这个选举过程非常简单粗暴，就是一个谁先谁当的过程，不涉及什么算法问题。

那成为 controller 之后要做啥呢，它会监听 zookeeper 里面的多个目录，例如有一个目录/brokers/，其他从节点往这个目录上**注册（就是往这个目录上创建属于自己的子目录而已）**自己，这时命名规则一般是它们的 id 编号，比如/brokers/0,1,2

注册时各个节点必定会暴露自己的主机名，端口号等等的信息，此时 controller 就要去**读取注册上来的从节点的数据（通过监听机制），生成集群的元数据信息，之后把这些信息都分发给其他的服务器，让其他服务器能感知到集群中其它成员的存在**。

此时模拟一个场景，我们创建一个主题（其实就是在 zookeeper 上/topics/topicA 这样创建一个目录而已），kafka 会把分区方案生成在这个目录中，此时 controller 就监听到了这一改变，它会去同步这个目录的元信息，然后同样下放给它的从节点，通过这个方法让整个集群都得知这个分区方案，此时从节点就各自创建好目录等待创建分区副本即可。这也是整个集群的管理机制。

## 三、加餐时间

### 1.Kafka 高性能

#### ① 顺序写

操作系统每次从磁盘读写数据的时候，需要先寻址，也就是先要找到数据在磁盘上的物理位置，然后再进行数据读写，如果是机械硬盘，寻址就需要较长的时间。\
kafka 的设计中，数据其实是存储在磁盘上面，一般来说，会把数据存储在内存上面性能才会好。但是 kafka 用的是顺序写，追加数据是追加到末尾，磁盘顺序写的性能极高，在磁盘个数一定，转数达到一定的情况下，基本和内存速度一致

随机写的话是在文件的某个位置修改数据，性能会较低。

#### ② 零拷贝

先来看看非零拷贝的情况

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_876152d4-2b45-11ec-b625-fa163eb4f6be.png)

可以看到数据的拷贝从内存拷贝到 kafka 服务进程那块，又拷贝到 socket 缓存那块，整个过程耗费的时间比较高，kafka 利用了 Linux 的 sendFile 技术（NIO），省去了进程切换和一次数据拷贝，让性能变得更好。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_87ae875c-2b45-11ec-b625-fa163eb4f6be.png)

### 2.日志分段存储

Kafka 规定了一个分区内的.log 文件最大为 1G，做这个限制目的是为了方便把.log 加载到内存去操作

```
00000000000000000000.index
00000000000000000000.log
00000000000000000000.timeindex

00000000000005367851.index
00000000000005367851.log
00000000000005367851.timeindex

00000000000009936472.index
00000000000009936472.log
00000000000009936472.timeindex
```

这个 9936472 之类的数字，就是代表了这个日志段文件里包含的起始 offset，也就说明这个分区里至少都写入了接近 1000 万条数据了。Kafka broker 有一个参数，log.segment.bytes，限定了每个日志段文件的大小，最大就是 1GB，一个日志段文件满了，就自动开一个新的日志段文件来写入，避免单个文件过大，影响文件的读写性能，这个过程叫做 log rolling，正在被写入的那个日志段文件，叫做 active log segment。

如果大家有看前面的两篇有关于 HDFS 的文章时，就会发现 NameNode 的 edits log 也会做出限制，所以这些框架都是会考虑到这些问题。

### 3.Kafka 的网络模型

kafka 的网络设计和 Kafka 的调优有关，这也是为什么它能支持高并发的原因

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_87ee9360-2b45-11ec-b625-fa163eb4f6be.png)

首先客户端发送请求全部会先发送给一个 Acceptor，broker 里面会存在 3 个线程（默认是 3 个），这 3 个线程都是叫做 processor，Acceptor 不会对客户端的请求做任何的处理，直接封装成一个个 socketChannel 发送给这些 processor 形成一个队列，发送的方式是轮询，就是先给第一个 processor 发送，然后再给第二个，第三个，然后又回到第一个。消费者线程去消费这些 socketChannel 时，会获取一个个 request 请求，这些 request 请求中就会伴随着数据。

线程池里面默认有 8 个线程，这些线程是用来处理 request 的，解析请求，如果 request 是写请求，就写到磁盘里。读的话返回结果。\
processor 会从 response 中读取响应数据，然后再返回给客户端。这就是 Kafka 的网络三层架构。

所以如果我们需要对 kafka 进行增强调优，增加 processor 并增加线程池里面的处理线程，就可以达到效果。request 和 response 那一块部分其实就是起到了一个缓存的效果，是考虑到 processor 们生成请求太快，线程数不够不能及时处理的问题。

所以这就是一个加强版的 reactor 网络线程模型。
