---
date created: 2023-03-08 14:05
---

#消息中间件 #分布式

## 一、分区一致性

在 Kafka 的生产者案例和消费者原理解析中我们提到 kafka 的内核里还有个 LEO&HW 原理，现在补充回来。

### LEO&HW 更新原理

首先这里有两个 Broker，也就是两台服务器，然后它们的分区中分别存储了两个 p0 的副本，一个是 leader，一个是 follower

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_6fcb4e5e-2b45-11ec-b625-fa163eb4f6be.png)

此时生产者往 leader partition 发送数据，数据最终肯定是要写到磁盘上的。然后 follower 会从 leader 那里去同步数据，follower 上的数据也会写到磁盘上

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_6fff3c32-2b45-11ec-b625-fa163eb4f6be.png)

可是 follower 是先从 leader 那里去同步再写入磁盘的，所以它磁盘上面的数据肯定会比 leader 的那块落后。

#### 1. LEO 是什么

LEO（last end offset）就是该副本底层日志文件上的数据的**最大偏移量的下一个值**，所以上图中 leader 那里的 LEO 就是 5+1 = 6，follower 的 LEO 是 5。以此类推，当我知道了 LEO 为 10，我就知道该日志文件已经保存了 10 条信息，位移范围为[0,9]

#### 2. HW 是什么

HW（highwater mark）就是水位，它一定会小于 LEO 的值。这个值规定了消费者仅能消费 HW 之前的数据。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_7042b282-2b45-11ec-b625-fa163eb4f6be.png)

#### 3. 流程分析

follower 在和 leader 同步数据的时候，同步过来的数据会带上 LEO 的值，可是在**实际情况中有可能 p0 的副本可能不仅仅只有 2 个**。此时我画多几个 follower(p0)，它们也向 leader partition 同步数据，带上自己的 LEO。**leader partition 就会记录这些 follower 同步过来的 LEO，然后取最小的 LEO 值作为 HW 值**

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_70754ad0-2b45-11ec-b625-fa163eb4f6be.png)

这个做法是保证了如果 leader partition 宕机，集群会从其它的 follower partition 里面选举出一个新的 leader partition。这时候无论选举了哪一个节点作为 leader，都能保证存在此刻待消费的数据，保证数据的安全性。

那么 follower 自身的 HW 的值如何确定，那就是**follower 获取数据时也带上 leader partition 的 HW 的值，然后和自身的 LEO 值取一个较小的值作为自身的 HW 值**。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_70b6992c-2b45-11ec-b625-fa163eb4f6be.png)

现在你再回想一下之前提到的 ISR，是不是就更加清楚了。**follower 如果超过 10 秒没有到 leader 这里同步数据，就会被踢出 ISR, 加入 OSR**。它的作用就是帮助我们在 leader 宕机时快速再选出一个 leader，因为在 ISR 列表中的 follower 都是和 leader 同步率高的，就算丢失数据也不会丢失太多。选举在 ISR 中的节点产生.

如果 OSR 中节点恢复同步, 则重新加入 ISR. 恢复的标准即: 当**follower 的 LEO 值>=leader 的 HW 值，就可以回到 ISR**。

可是按照刚刚的流程确实无法避免丢失部分数据的情况，当然也是有办法来保证数据的完整的，咱们留到源码篇之后进行总结的时候再提。

#### 4. 总结

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_70ee2a5e-2b45-11ec-b625-fa163eb4f6be.png)

## 二、Kafka 的流程梳理

首先来两个 Broker（这集群好歹要超过 1 个服务器才能叫集群吧），然后它们启动的时候会往 zookeeper 集群中注册，这时候这两台服务器会抢占一个名字叫 controller 的目录，谁抢到了，谁就是 controller。比如现在第一台 Broker 抢到了。那它就是**controller，它要监听 zookeeper 中各个目录的变化，管理整个集群的元数据**。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_712be70e-2b45-11ec-b625-fa163eb4f6be.png)

此时我们通过客户端来用命令来创建一个主题，这时候会有一个主题的分区方案写入到 zookeeper 的目录中，而在 controller 监听到这个目录写入了分区方案（其实就是一些元数据信息）之后，它也会更改自己的元数据信息。之后其他的 Broker 也会向 controller 来同步元数据。保证整个集群的 Broker 的元数据都是一致的

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_717d0ecc-2b45-11ec-b625-fa163eb4f6be.png)

此时再比如我们现在通过元数据信息得知有一个分区 p0，leader partition 在第一台 Broker，follower partition 在第二台 Broker。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_71adccba-2b45-11ec-b625-fa163eb4f6be.png)

此时生产者就该出来了，生产者需要往集群发送消息前，要先把每一条消息封装成 ProducerRecord 对象，这是生产者内部完成的。之后会经历一个序列化的过程。接下来它需要过去集群中拉取元数据（所以大家知道为啥在 插曲：Kafka 的生产者原理及重要参数说明 的 1-⑤-1 生产者代码里面为啥要提供一个或多个 broker 的地址了吧），当时的代码片段如下

```java
props.put("bootstrap.servers", "hadoop1:9092,hadoop2:9092,hadoop3:9092");
```

因为如果不提供服务器的地址，是没法获取到元数据信息的。此时生产者的消息是不知道该发送给哪个服务器的哪个分区的。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_71d9a5b0-2b45-11ec-b625-fa163eb4f6be.png)

此时生产者不着急把消息发送出去，而是先放到一个缓冲区。把消息放进缓冲区之后，与此同时会有一个独立线程 Sender 去把消息分批次包装成一个个 Batch(先攒后发)。整好一个个 batch 之后，就开始发送给对应的主机上面。此时经过 大白话篇 中加餐时间所提到的**Kafka 的三层网络架构模型(IO 多路复用)**，写到 os cache，再继续写到磁盘上面。

之后写磁盘的过程又要将 Kafka 的生产者案例和消费者原理解析 中提到的**日志二分查找**，和刚刚才提完的**ISR,LEO 和 HW**。因为当 leader 写入完成时，follower 又要过去同步数据了。

此时消费者组也进来，这个消费者组会有一个它们的 group.id 号，根据这个可以计算出哪一个 broker 作为它们的 coodinator，确定了 coordinator 之后，所有的 consumer 都会发送一个 join group 请求注册。之后**coordinator 就会默认把第一个注册上来的 consumer 选择成为 leader consumer**，**把整个 Topic 的情况汇报给 leader consumer。之后 leader consumer 就会根据负载均衡的思路制定消费方案，返回给 coordinator，coordinator 拿到方案之后再下发给所有的 consumer**，完成流程。

![](https://oss-emcsprod-public.modb.pro/wechatSpider/modb_20211012_720cd57a-2b45-11ec-b625-fa163eb4f6be.png)
