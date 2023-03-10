## seata

> 以下内容，来自  [官网](https://seata.io/zh-cn/)

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。

### seata 简介

Seata , [官网](https://seata.io/zh-cn/) , [github](https://github.com/seata/seata) , 1 万多星

- Seata 是一款开源的分布式事务解决方案，致力于在微服务架构下提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式
- 在 Seata 开源之前，Seata 对应的内部版本在阿里经济体内部一直扮演着分布式一致性中间件的角色，帮助经济体平稳的度过历年的双 11，对各 BU 业务进行了有力的支撑。商业化产品[GTS](https://www.aliyun.com/aliware/txc?spm=5176.8142029.388261.386.a72376f4lqvQxv)  先后在阿里云、金融云进行售卖

**相关链接：**

**什么是 seata：**[https://seata.io/zh-cn/docs/overview/what-is-seata.html](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fseata.io%2Fzh-cn%2Fdocs%2Foverview%2Fwhat-is-seata.html)

**下载** [https://seata.io/zh-cn/blog/download.html](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fseata.io%2Fzh-cn%2Fblog%2Fdownload.html)  
**官方例子** [https://seata.io/zh-cn/docs/user/quickstart.html](https://www.oschina.net/action/GoToLink?url=https%3A%2F%2Fseata.io%2Fzh-cn%2Fdocs%2Fuser%2Fquickstart.html)

### Seata 的三大模块

Seata AT 使用了增强型二阶段提交实现。

Seata 分三大模块 :

- TC ：事务协调者。负责我们的事务 ID 的生成，事务注册、提交、回滚等。
- TM：事务发起者。定义事务的边界，负责告知 TC，分布式事务的开始，提交，回滚。
- RM：资源管理者。管理每个分支事务的资源，每一个 RM 都会作为一个分支事务注册在 TC。

在 Seata 的 AT 模式中，TM 和 RM 都作为 SDK 的一部分和业务服务在一起，我们可以认为是 Client。TC 是一个独立的服务，通过服务的注册、发现将自己暴露给 Client 们。

Seata 中有三大模块中， TM 和 RM 是作为 Seata 的客户端与业务系统集成在一起，TC 作为 Seata 的服务端独立部署。

![640?wx_fmt=png](https://mmbiz.qpic.cn/mmbiz_png/nibOZpaQKw09icwew0IKibawDpFiaiaoQyVnZib6tic9rNAicNn6I6LhuSHs31Kww2jJv1YibCvbkUqN0OzZYk4b17TytyQ/640?wx_fmt=png)

### Seata 的分布式事务的执行流程

在 Seata 中，分布式事务的执行流程：

- TM 开启分布式事务（TM 向 TC 注册全局事务记录）；
- 按业务场景，编排数据库、服务等事务内资源（RM 向 TC 汇报资源准备状态 ）；
- TM 结束分布式事务，事务一阶段结束（TM 通知 TC 提交/回滚分布式事务）；
- TC 汇总事务信息，决定分布式事务是提交还是回滚；
- TC 通知所有 RM 提交/回滚 资源，事务二阶段结束；

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210710161530943.png)

Seata 的 TC、TM、RM 三个角色 ， 是不是和 XA 模型很像. 下图是 XA 模型的事务大致流程。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210710161627404.png)

在 X/Open **DTP(Distributed Transaction Process)**模型里面，有三个角色：

​ AP: Application，应用程序。也就是业务层。哪些操作属于一个事务，就是 AP 定义的。

​ TM: Transaction Manager，事务管理器。接收 AP 的事务请求，对全局事务进行管理，管理事务分支状态，协调 RM 的处理，通知 RM 哪些操作属于哪些全局事务以及事务分支等等。这个也是整个事务调度模型的核心部分。

​ RM：Resource Manager，资源管理器。一般是数据库，也可以是其他的资源管理器，如消息队列(如 JMS 数据源)，文件系统等。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210709140731779.png)

### 4 种分布式事务解决方案

Seata 会有 4 种分布式事务解决方案，分别是 AT 模式、TCC 模式、Saga 模式和 XA 模式。

![640?wx_fmt=jpeg](https://mmbiz.qpic.cn/mmbiz_jpg/nibOZpaQKw09icwew0IKibawDpFiaiaoQyVnZpWibNo9daxIshiaduyW2AwqkFx9dXEPNicyKwMicZdg41aStokBZhB0E8Q/640?wx_fmt=jpeg)

## Seata AT 模式

Seata AT 模式是最早支持的模式。AT 模式是指**Automatic (Branch) Transaction Mode**自动化分支事务。

Seata AT 模式是增强型 2pc 模式，或者说是增强型的 XA 模型。

总体来说，AT 模式，是 2pc 两阶段提交协议的演变，不同的地方，Seata AT 模式不会一直锁表。

### Seata AT 模式的使用前提

- 基于支持本地 ACID 事务的关系型数据库。

  > 比如，在 MySQL 5.1 之前的版本中，默认的搜索引擎是 MyISAM，从 MySQL 5.5 之后的版本中，默认的搜索引擎变更为 InnoDB。MyISAM 存储引擎的特点是：表级锁、不支持事务和全文索引。 所以，基于 MyISAM 的表，就不支持 Seata AT 模式。

- Java 应用，通过 JDBC 访问数据库。

### Seata AT 模型图

两阶段提交协议的演变：

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
- 二阶段：
  - 提交异步化，非常快速地完成。
  - 或回滚通过一阶段的回滚日志进行反向补偿

完整的 AT 在 Seata 所制定的事务模式下的模型图：  
![img](https://img-blog.csdnimg.cn/20201009172509465.png)

### Seata AT 模式的例子

我们用一个比较简单的业务场景来描述一下 Seata AT 模式的工作过程。

有个充值业务，现在有两个服务，一个负责管理用户的余额，另外一个负责管理用户的积分。

> 当用户充值的时候，首先增加用户账户上的余额，然后增加用户的积分。

Seata AT 分为两阶段，主要逻辑全部在第一阶段，第二阶段主要做回滚或日志清理的工作。

#### 第一阶段流程

第一阶段流程如

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117211026990.png)

1）余额服务中的 TM，向 TC 申请开启一个全局事务，TC 会返回一个全局的事务 ID。

2）余额服务在执行本地业务之前，RM 会先向 TC 注册分支事务。

3）余额服务依次生成 undo log、执行本地事务、生成 redo log，最后直接提交本地事务。

4）余额服务的 RM 向 TC 汇报，事务状态是成功的。

5）余额服务发起远程调用，把事务 ID 传给积分服务。

6）积分服务在执行本地业务之前，也会先向 TC 注册分支事务。

7）积分服务次生成 undo log、执行本地事务、生成 redo log，最后直接提交本地事务。

8）积分服务的 RM 向 TC 汇报，事务状态是成功的。

9）积分服务返回远程调用成功给余额服务。

10）余额服务的 TM 向 TC 申请全局事务的提交/回滚。

> 积分服务中也有 TM，但是由于没有用到，因此直接可以忽略。

我们如果使用 Spring 框架的注解式事务，远程调用会在本地事务提交之前发生。但是，先发起远程调用还是先提交本地事务，这个其实没有任何影响。

#### 第二阶段流程如

第二阶段的逻辑就比较简单了。

Client 和 TC 之间是有长连接的，如果是正常全局提交，则 TC 通知多个 RM 异步清理掉本地的 redo 和 undo log 即可。如果是回滚，则 TC 通知每个 RM 回滚数据即可。

这里就会引出一个问题，由于本地事务都是自己直接提交了，后面如何回滚，由于我们在操作本地业务操作的前后，做记录了 undo 和 redo log，因此可以通过 undo log 进行回滚。

由于 undo 和 redo log 和业务操作在同一个事务中，因此肯定会同时成功或同时失败。

但是还会存在一个问题，因为每个事务从本地提交到通知回滚这段时间里，可能这条数据已经被别的事务修改，如果直接用 undo log 回滚，会导致数据不一致的情况。

此时，RM 会用 redo log 进行校验，对比数据是否一样，从而得知数据是否有别的事务修改过。注意：undo log 是被修改前的数据，可以用于回滚；redo log 是被修改后的数据，用于回滚校验。

如果数据未被其他事务修改过，则可以直接回滚；如果是脏数据，再根据不同策略处理。

### Seata AT 模式在电商下单场景的使用

下面描述 Seata AT mode 的工作原理使用的电商下单场景的使用

如下图所示：

![img](https://img-blog.csdnimg.cn/20210711210909395.png)

在上图中，协调者 shopping-service 先调用参与者 repo-service 扣减库存，后调用参与者 order-service 生成订单。这个业务流使用 Seata in XA mode 后的全局事务流程如下图所示：

![img](https://img-blog.csdnimg.cn/20210711210932176.png)

上图描述的全局事务执行流程为：

1）shopping-service 向 Seata 注册全局事务，并产生一个全局事务标识 XID

2）将 repo-service.repo_db、order-service.order_db 的本地事务执行到待提交阶段，事务内容包含对 repo-service.repo_db、order-service.order_db 进行的查询操作以及写每个库的 undo_log 记录

3）repo-service.repo_db、order-service.order_db 向 Seata 注册分支事务，并将其纳入该 XID 对应的全局事务范围

4）提交 repo-service.repo_db、order-service.order_db 的本地事务

5）repo-service.repo_db、order-service.order_db 向 Seata 汇报分支事务的提交状态

6）Seata 汇总所有的 DB 的分支事务的提交状态，决定全局事务是该提交还是回滚

7）Seata 通知 repo-service.repo_db、order-service.order_db 提交/回滚本地事务，若需要回滚，采取的是补偿式方法

其中 1）2）3）4）5）属于第一阶段，6）7）属于第二阶段。

#### 电商业务场景中 Seata in AT mode 工作流程详述

在上面的电商业务场景中，购物服务调用库存服务扣减库存，调用订单服务创建订单，显然这两个调用过程要放在一个事务里面。即：

```sql
start global_trx

 call 库存服务的扣减库存接口

 call 订单服务的创建订单接口

commit global_trx
```

在库存服务的数据库中，存在如下的库存表 t_repo：

![img](https://img-blog.csdnimg.cn/20210711210955548.png)

在订单服务的数据库中，存在如下的订单表 t_order：

![img](https://img-blog.csdnimg.cn/20210711211013809.png)

现在，id 为 40002 的用户要购买一只商品代码为 20002 的鼠标，整个分布式事务的内容为：

1）在库存服务的库存表中将记录

![img](https://img-blog.csdnimg.cn/20210711211030835.png)

修改为

![img](https://img-blog.csdnimg.cn/20210711211044143.png)

2）在订单服务的订单表中添加一条记录

![img](https://img-blog.csdnimg.cn/20210711211102524.png)

以上操作，在 AT 模式的第一阶段的流程图如下：

![img](https://img-blog.csdnimg.cn/20210711211149471.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210711211222402.png)

从 AT 模式第一阶段的流程来看，分支的本地事务在第一阶段提交完成之后，就会释放掉本地事务锁定的本地记录。这是 AT 模式和 XA 最大的不同点，在 XA 事务的两阶段提交中，被锁定的记录直到第二阶段结束才会被释放。所以 AT 模式减少了锁记录的时间，从而提高了分布式事务的处理效率。AT 模式之所以能够实现第一阶段完成就释放被锁定的记录，是因为 Seata 在每个服务的数据库中维护了一张 undo_log 表，其中记录了对 t_order / t_repo 进行操作前后记录的镜像数据，即便第二阶段发生异常，只需回放每个服务的 undo_log 中的相应记录即可实现全局回滚。

undo_log 的表结构：

![img](https://img-blog.csdnimg.cn/20210711211250507.png)

第一阶段结束之后，Seata 会接收到所有分支事务的提交状态，然后决定是提交全局事务还是回滚全局事务。

1）若所有分支事务本地提交均成功，则 Seata 决定全局提交。Seata 将分支提交的消息发送给各个分支事务，各个分支事务收到分支提交消息后，会将消息放入一个缓冲队列，然后直接向 Seata 返回提交成功。之后，每个本地事务会慢慢处理分支提交消息，处理的方式为：删除相应分支事务的 undo_log 记录。之所以只需删除分支事务的 undo_log 记录，而不需要再做其他提交操作，是因为提交操作已经在第一阶段完成了（这也是 AT 和 XA 不同的地方）。这个过程如下图所示：

![img](https://img-blog.csdnimg.cn/20210711211745801.png)

分支事务之所以能够直接返回成功给 Seata，是因为真正关键的提交操作在第一阶段已经完成了，清除 undo_log 日志只是收尾工作，即便清除失败了，也对整个分布式事务不产生实质影响。

2）若任一分支事务本地提交失败，则 Seata 决定全局回滚，将分支事务回滚消息发送给各个分支事务，由于在第一阶段各个服务的数据库上记录了 undo_log 记录，分支事务回滚操作只需根据 undo_log 记录进行补偿即可。全局事务的回滚流程如下图所示：

![img](https://img-blog.csdnimg.cn/20210711211811749.png)

这里对图中的 2、3 步做进一步的说明：

1）由于上文给出了 undo_log 的表结构，所以可以通过 xid 和 branch_id 来找到当前分支事务的所有 undo_log 记录；

2）拿到当前分支事务的 undo_log 记录之后，首先要做数据校验，如果 afterImage 中的记录与当前的表记录不一致，说明从第一阶段完成到此刻期间，有别的事务修改了这些记录，这会导致分支事务无法回滚，向 Seata 反馈回滚失败；如果 afterImage 中的记录与当前的表记录一致，说明从第一阶段完成到此刻期间，没有别的事务修改这些记录，分支事务可回滚，进而根据 beforeImage 和 afterImage 计算出补偿 SQL，执行补偿 SQL 进行回滚，然后删除相应 undo_log，向 Seata 反馈回滚成功。

### Seata 的数据隔离性

seata 的 at 模式主要实现逻辑是数据源代理，而数据源代理将基于如 MySQL 和 Oracle 等关系事务型数据库实现，基于数据库的隔离级别为 read committed。换而言之，本地事务的支持是 seata 实现 at 模式的必要条件，这也将限制 seata 的 at 模式的使用场景。

#### 写隔离

从前面的工作流程，我们可以很容易知道，Seata 的写隔离级别是全局独占的。

首先，我们理解一下写隔离的流程

```markdown
分支事务 1-开始
|
V 获取 本地锁
|
V 获取 全局锁 分支事务 2-开始
| |
V 释放 本地锁 V 获取 本地锁
| |
V 释放 全局锁 V 获取 全局锁
|
V 释放 本地锁
|
V 释放 全局锁
```

如上所示，一个分布式事务的锁获取流程是这样的  
1）先获取到本地锁，这样你已经可以修改本地数据了，只是还不能本地事务提交  
2）而后，能否提交就是看能否获得全局锁  
3）获得了全局锁，意味着可以修改了，那么提交本地事务，释放本地锁  
4）当分布式事务提交，释放全局锁。这样就可以让其它事务获取全局锁，并提交它们对本地数据的修改了。

可以看到，这里有两个关键点  
1）本地锁获取之前，不会去争抢全局锁  
2）全局锁获取之前，不会提交本地锁

这就意味着，数据的修改将被互斥开来。也就不会造成写入脏数据。全局锁可以让分布式修改中的写数据隔离。

##### 写隔离的原则

- 一阶段本地事务提交前，需要确保先拿到  **全局锁** 。
- 拿不到  **全局锁** ，不能提交本地事务。
- 拿  **全局锁**  的尝试被限制在一定范围内，超出范围将放弃，并回滚本地事务，释放本地锁。

以一个示例来说明：

> 两个全局事务 tx1 和 tx2，分别对 a 表的 m 字段进行更新操作，m 的初始值 1000。

tx1 先开始，开启本地事务，拿到本地锁，更新操作 m = 1000 - 100 = 900。本地事务提交前，先拿到该记录的  **全局锁** ，本地提交释放本地锁。

tx2 后开始，开启本地事务，拿到本地锁，更新操作 m = 900 - 100 = 800。本地事务提交前，尝试拿该记录的  **全局锁** ，tx1 全局提交前，该记录的全局锁被 tx1 持有，tx2 需要重试等待  **全局锁** 。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210710164507421.png)

tx1 二阶段全局提交，释放  **全局锁** 。tx2 拿到  **全局锁**  提交本地事务。

如果 tx1 的二阶段全局回滚，则 tx1 需要重新获取该数据的本地锁，进行反向补偿的更新操作，实现分支的回滚。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021071016463052.png)

此时，如果 tx2 仍在等待该数据的  **全局锁**，同时持有本地锁，则 tx1 的分支回滚会失败。分支的回滚会一直重试，直到 tx2 的  **全局锁**  等锁超时，放弃  **全局锁**  并回滚本地事务释放本地锁，tx1 的分支回滚最终成功。

因为整个过程  **全局锁**  在 tx1 结束前一直是被 tx1 持有的，所以不会发生  **脏写**  的问题。

#### 读的隔离级别

在数据库本地事务隔离级别  **读已提交（Read Committed）**  或以上的基础上，Seata（AT 模式）的默认全局隔离级别是  **读未提交（Read Uncommitted）** 。

如果应用在特定场景下，必需要求全局的  **读已提交** ，目前 Seata 的方式是通过 SELECT FOR UPDATE 语句的代理。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210710165141349.png)

SELECT FOR UPDATE 语句的执行会申请  **全局锁** ，如果  **全局锁**  被其他事务持有，则释放本地锁（回滚 SELECT FOR UPDATE 语句的本地执行）并重试。这个过程中，查询是被 block 住的，直到  **全局锁**  拿到，即读取的相关数据是  **已提交**  的，才返回。

出于总体性能上的考虑，Seata 目前的方案并没有对所有 SELECT 语句都进行代理，仅针对 FOR UPDATE 的 SELECT 语句。

### Spring Cloud 集成 Seata AT 模式

AT 模式是指**Automatic (Branch) Transaction Mode 自动化分支事务，**使用 AT 模式的前提是

- 基于支持本地 ACID 事务的关系型数据库。
- Java 应用，通过 JDBC 访问数据库。

**seata-at 的使用步骤**

1、引入 seata 框架，配置好 seata 基本配置，建立 undo_log 表

2、消费者引入全局事务注解@GlobalTransactional

3、生产者引入全局事务注解@GlobalTransactional

> 此处没有写完， 尼恩的博文，都是迭代模式，后续会持续

## Seata TCC 模式

### 简介

TCC 与 Seata AT 事务一样都是**两阶段事务**，它与 AT 事务的主要区别为：

- **TCC 对业务代码侵入严重**  
  每个阶段的数据操作都要自己进行编码来实现，事务框架无法自动处理。
- **TCC 性能更高**  
  不必对数据加**全局锁**，允许多个事务同时操作数据。

  ![a](https://img-blog.csdnimg.cn/img_convert/ecaeda572495e4ea5e308fb938a38fd5.png)

Seata TCC 整体是  **两阶段提交**  的模型。一个分布式的全局事务，全局事务是由若干分支事务组成的，分支事务要满足  **两阶段提交**  的模型要求，即需要每个分支事务都具备自己的：

- 一阶段 prepare 行为
- 二阶段 commit 或 rollback 行为

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210710224912234.png)

根据两阶段行为模式的不同，我们将分支事务划分为  **Automatic (Branch) Transaction Mode**  和  **TCC (Branch) Transaction Mode**.

AT 模式（[参考链接 TBD](https://seata.io/zh-cn/docs/dev/mode/tcc-mode.html)）基于  **支持本地 ACID 事务**  的  **关系型数据库**：

- 一阶段 prepare 行为：在本地事务中，一并提交业务数据更新和相应回滚日志记录。
- 二阶段 commit 行为：马上成功结束，**自动**  异步批量清理回滚日志。
- 二阶段 rollback 行为：通过回滚日志，**自动**  生成补偿操作，完成数据回滚。

相应的，TCC 模式，不依赖于底层数据资源的事务支持：

- 一阶段 prepare 行为：调用  **自定义**  的 prepare 逻辑。
- 二阶段 commit 行为：调用  **自定义**  的 commit 逻辑。
- 二阶段 rollback 行为：调用  **自定义**  的 rollback 逻辑。

所谓 TCC 模式，是指支持把  **自定义**  的分支事务纳入到全局事务的管理中。

#### 第一阶段 Try

以账户服务为例，当下订单时要扣减用户账户金额：

![a](https://img-blog.csdnimg.cn/img_convert/6223e8be682d613f39e5a27587635f73.png)

假如用户购买 100 元商品，要扣减 100 元。

TCC 事务首先对这 100 元的扣减金额进行预留，或者说是先冻结这 100 元：

![a](https://img-blog.csdnimg.cn/img_convert/29586d0d5b4a08e27f784a80a450e9a5.png)

#### 第二阶段 Confirm

如果第一阶段能够顺利完成，那么说明“扣减金额”业务(分支事务)最终肯定是可以成功的。当全局事务提交时， TC 会控制当前分支事务进行提交，如果提交失败，TC 会反复尝试，直到提交成功为止。

当全局事务提交时，就可以使用冻结的金额来最终实现业务数据操作：

![a](https://img-blog.csdnimg.cn/img_convert/6227e7939dc860283ea799abcdccbe2b.png)

#### 第二阶段 Cancel

如果全局事务回滚，就把冻结的金额进行解冻，恢复到以前的状态，TC 会控制当前分支事务回滚，如果回滚失败，TC 会反复尝试，直到回滚完成为止。

![a](https://img-blog.csdnimg.cn/img_convert/ea3d9416a1db9998241c160b67078c51.png)

#### 多个事务并发的情况

多个 TCC 全局事务允许并发，它们执行扣减金额时，只需要冻结各自的金额即可：

![a](https://img-blog.csdnimg.cn/img_convert/5c78e69b171903aaef07621076019fba.png)

> Seata TCC 模式 没有写完， 尼恩的博文，都是迭代模式，后续会持续优化

## SEATA Saga 模式

Saga 模式是 SEATA 提供的长事务解决方案，在 Saga 模式中，业务流程中每个参与者都提交本地事务，当出现某一个参与者失败则补偿前面已经成功的参与者，一阶段正向服务和二阶段补偿服务都由业务开发实现。

![Saga模式示意图](https://img.alicdn.com/tfs/TB1Y2kuw7T2gK0jSZFkXXcIQFXa-445-444.png)

理论基础：Hector & Kenneth 发表论⽂ Sagas （1987）

### 适用场景

- 业务流程长、业务流程多
- 参与者包含其它公司或遗留系统服务，无法提供 TCC 模式要求的三个接口

### 优势

- 一阶段提交本地事务，无锁，高性能
- 事件驱动架构，参与者可异步执行，高吞吐
- 补偿服务易于实现

### 缺点

- 不保证隔离性（应对方案见后面文档）

### Saga 的实现

#### 基于状态机引擎的 Saga 实现

目前 SEATA 提供的 Saga 模式是基于状态机引擎来实现的，机制是：

1. 通过状态图来定义服务调用的流程并生成 json 状态语言定义文件
2. 状态图中一个节点可以是调用一个服务，节点可以配置它的补偿节点
3. 状态图 json 由状态机引擎驱动执行，当出现异常时状态引擎反向执行已成功节点对应的补偿节点将事务回滚

> 注意: 异常发生时是否进行补偿也可由用户自定义决定

1. 可以实现服务编排需求，支持单项选择、并发、子流程、参数转换、参数映射、服务执行状态判断、异常捕获等功能

示例状态图:

![示例状态图](https://seata.io/img/saga/demo_statelang.png?raw=true)

> Seata Saga 模式 没有写完， 尼恩的博文，都是迭代模式，后续会持续优化

## Seata XA 模式

### 使用 Seata XA 模式的前提

- 支持 XA 事务的数据库。
- Java 应用，通过 JDBC 访问数据库。

### Seata XA 模式的整体机制

在 Seata 定义的分布式事务框架内，利用事务资源（数据库、消息服务等）对 XA 协议的支持，以 XA 协议的机制来管理分支事务的一种 事务模式。

> 注意这里的重点：利用事务资源对 XA 协议的支持，以 XA 协议的机制来管理分支事务。

![img](https://img.alicdn.com/tfs/TB1hSpccIVl614jSZKPXXaGjpXa-1330-924.png)

### Seata XA 模式的工作机制

#### 1. 整体运行机制

XA 模式 运行在 Seata 定义的事务框架内：

![xa-fw](https://img.alicdn.com/tfs/TB1uM2OaSslXu8jSZFuXXXg7FXa-1330-958.png)

#### 2. 数据源代理

XA 模式需要 XAConnection。

获取 XAConnection 两种方式：

- 方式一：要求开发者配置 XADataSource
- 方式二：根据开发者的普通 DataSource 来创建

第一种方式，给开发者增加了认知负担，需要为 XA 模式专门去学习和使用 XA 数据源，与 透明化 XA 编程模型的设计目标相违背。

第二种方式，对开发者比较友好，和 AT 模式使用一样，开发者完全不必关心 XA 层面的任何问题，保持本地编程模型即可。

我们优先设计实现第二种方式：数据源代理根据普通数据源中获取的普通 JDBC 连接创建出相应的 XAConnection。

类比 AT 模式的数据源代理机制，如下：

![ds1](https://img.alicdn.com/tfs/TB11_LJcggP7K4jSZFqXXamhVXa-1564-894.png)

实际上，这种方法是在做数据库驱动程序要做的事情。不同的厂商、不同版本的数据库驱动实现机制是厂商私有的，我们只能保证在充分测试过的驱动程序上是正确的，开发者使用的驱动程序版本差异很可能造成机制的失效。

这点在 Oracle 上体现非常明显。参见 Druid issue：[https://github.com/alibaba/druid/issues/3707](https://github.com/alibaba/druid/issues/3707)

综合考虑，XA 模式的数据源代理设计需要同时支持第一种方式：基于 XA 数据源进行代理。

类比 AT 模式的数据源代理机制，如下：

![ds2](https://img.alicdn.com/tfs/TB1qJ57XZieb18jSZFvXXaI3FXa-1564-894.png)

XA start 需要 Xid 参数。

这个 Xid 需要和 Seata 全局事务的 XID 和 BranchId 关联起来，以便由 TC 驱动 XA 分支的提交或回滚。

目前 Seata 的 BranchId 是在分支注册过程，由 TC 统一生成的，所以 XA 模式分支注册的时机需要在 XA start 之前。

将来一个可能的优化方向：

把分支注册尽量延后。类似 AT 模式在本地事务提交之前才注册分支，避免分支执行失败情况下，没有意义的分支注册。

这个优化方向需要 BranchId 生成机制的变化来配合。BranchId 不通过分支注册过程生成，而是生成后再带着 BranchId 去注册分支。

### XA 模式的使用

从编程模型上，XA 模式与 AT 模式保持完全一致。

可以参考 Seata 官网的样例：[seata-xa](https://github.com/seata/seata-samples/tree/master/seata-xa)

样例场景是 Seata 经典的，涉及库存、订单、账户 3 个微服务的商品订购业务。

在样例中，上层编程模型与 AT 模式完全相同。只需要修改数据源代理，即可实现 XA 模式与 AT 模式之间的切换。

```java
    @Bean("dataSource")
    public DataSource dataSource(DruidDataSource druidDataSource) {
        // DataSourceProxy for AT mode
        // return new DataSourceProxy(druidDataSource);

        // DataSourceProxyXA for XA mode
        return new DataSourceProxyXA(druidDataSource);
    }
```

## Seata 实践

Seata 是一个需独立部署的中间件，所以先搭 Seata Server，这里以最新的  `seata-server-1.4.0`  版本为例，下载地址：`https://seata.io/en-us/blog/download.html`

解压后的文件我们只需要关心  `\seata\conf`  目录下的  `file.conf`  和  `registry.conf`  文件。

### Seata Server

#### file.conf

`file.conf`  文件用于配置持久化事务日志的模式，目前提供  `file`、`db`、`redis`  三种方式。

![file.conf 文件配置](https://img-blog.csdnimg.cn/2020111711164266.png?#pic_center)

**注意**：在选择  `db`  方式后，需要在对应数据库创建  `globalTable`（持久化全局事务）、`branchTable`（持久化各提交分支的事务）、 `lockTable`（持久化各分支锁定资源事务）三张表。

```sql
-- the table to store GlobalSession data
-- 持久化全局事务
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store BranchSession data
-- 持久化各提交分支的事务
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store lock data
-- 持久化每个分支锁表事务
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(96),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

#### registry.conf

`registry.conf`  文件设置 注册中心 和 配置中心：

目前注册中心支持  `nacos` 、`eureka`、`redis`、`zk`、`consul`、`etcd3`、`sofa`  七种，这里我使用的  `eureka`作为注册中心 ； 配置中心支持  `nacos` 、`apollo`、`zk`、`consul`、`etcd3`  五种方式。

![registry.conf 文件配置](https://img-blog.csdnimg.cn/20201117104729247.png?#pic_center)

配置完以后在  `\seata\bin`  目录下启动  `seata-server`  即可，到这  `Seata`  的服务端就搭建好了。

### Seata Client

`Seata Server`  环境搭建完，接下来我们新建三个服务  `order-server`（下单服务）、`storage-server`（扣减库存服务）、`account-server`（账户金额服务），分别服务注册到  `eureka`。

每个服务的大体核心配置如下：

```yml
spring:
  application:
    name: storage-server
  cloud:
    alibaba:
      seata:
        tx-service-group: my_test_tx_group
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://47.93.6.1:3306/seat-storage
    username: root
    password: root

# eureka 注册中心
eureka:
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:8761/eureka/
  instance:
    hostname: 47.93.6.5
    prefer-ip-address: true
```

业务大致流程：用户发起下单请求，本地 order 订单服务创建订单记录，并通过  `RPC`  远程调用  `storage`  扣减库存服务和  `account`  扣账户余额服务，只有三个服务同时执行成功，才是一个完整的下单流程。如果某个服执行失败，则其他服务全部回滚。

Seata 对业务代码的侵入性非常小，代码中使用只需用  `@GlobalTransactional`  注解开启一个全局事务即可。

```java
@Override
@GlobalTransactional(name = "create-order", rollbackFor = Exception.class)
public void create(Order order) {

    String xid = RootContext.getXID();

    LOGGER.info("------->交易开始");
    //本地方法
    orderDao.create(order);

    //远程方法 扣减库存
    storageApi.decrease(order.getProductId(), order.getCount());

    //远程方法 扣减账户余额
    LOGGER.info("------->扣减账户开始order中");
    accountApi.decrease(order.getUserId(), order.getMoney());
    LOGGER.info("------->扣减账户结束order中");

    LOGGER.info("------->交易结束");
    LOGGER.info("全局事务 xid： {}", xid);
}
```

前边说过 Seata AT 模式实现分布式事务，必须在相关的业务库中创建  `undo_log`  表来存数据回滚日志，表结构如下：

```sql
-- for AT mode you must to init this sql for you business database. the seata server not need it.
CREATE TABLE IF NOT EXISTS `undo_log`
(
    `id`            BIGINT(20)   NOT NULL AUTO_INCREMENT COMMENT 'increment id',
    `branch_id`     BIGINT(20)   NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(100) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME     NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME     NOT NULL COMMENT 'modify datetime',
    PRIMARY KEY (`id`),
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8 COMMENT ='AT transaction mode undo table';
```

> 到这环境搭建的工作就完事了，完整案例会在后边贴出  `GitHub`  地址，就不在这占用篇幅了。

## 测试 Seata

项目中的服务调用过程如下图：

![服务调用过程](https://img-blog.csdnimg.cn/20201119195532937.png?#pic_center)

启动各个服务后，我们直接请求下单接口看看效果，只要  `order`  订单表创建记录成功，`storage`  库存表  `used`  字段数量递增、`account`  余额表  `used`  字段数量递增则表示下单流程成功。

![原始数据](https://img-blog.csdnimg.cn/20201124180539441.png?#pic_center)

请求后正向流程是没问题的，数据和预想的一样

![下单数据](https://img-blog.csdnimg.cn/2020112418131572.png?#pic_center)

而且发现  `TM`  事务管理者  `order-server`  服务的控制台也打印出了两阶段提交的日志

![控制台两次提交](https://img-blog.csdnimg.cn/20201124195638183.png?#pic_center)

那么再看看如果其中一个服务异常，会不会正常回滚呢？在  `account-server`  服务中模拟超时异常，看能否实现全局事务回滚。

![全局事务回滚](https://img-blog.csdnimg.cn/20201124202402544.png?#pic_center)

发现数据全没执行成功，说明全局事务回滚也成功了

![](https://img-blog.csdnimg.cn/20201124203042277.png?#pic_center)

那看一下  `undo_log`  回滚记录表的变化情况，由于  `Seata`  删除回滚日志的速度很快，所以要想在表中看见回滚日志，必须要在某一个服务上打断点才看的更明显。

![回滚记录](https://img-blog.csdnimg.cn/20201116192806226.png#pic_center)

## 总结

上边简单介绍了  `2PC`、`3PC`、`TCC`、`MQ`、`Seata`  这五种分布式事务解决方案，还详细的实践了  `Seata`  中间件。但不管我们选哪一种方案，在项目中应用都要谨慎再谨慎，除特定的数据强一致性场景外，能不用尽量就不要用，因为无论它们性能如何优越，一旦项目套上分布式事务，整体效率会几倍的下降，在高并发情况下弊端尤为明显。

> 本案例 github 地址：[https://github.com/chengxy-nds/Springboot-Notebook/tree/master/springboot-seata-transaction](https://github.com/chengxy-nds/Springboot-Notebook/tree/master/springboot-seata-transaction)

## 面试题标准答案: 如何解决分布式事务问题的？

现在 Java 面试，分布式系统、分布式事务几乎是标配。而分布式系统、分布式事务本身比较复杂，大家学起来也非常头疼。

```undefined
 面试题：分布式事务了解吗？你们是如何解决分布式事务问题的？
```

Seata AT 模式和 Seata TCC 是在生产中最常用。

- 强一致性模型，Seata AT **强一致方案**  模式用于强一致主要用于核心模块，例如交易/订单等。
- 弱一致性模型。Seata TCC **弱一致方案**一般用于边缘模块例如库存，通过 TC 的协调，保证最终一致性，也可以业务解耦。

> 面试中如果你真的被问到，可以分场景回答：

### （1）强一致性场景

对于那些特别严格的场景，用的是 Seata AT 模式来保证强一致性；

准备好例子：你找一个严格要求数据绝对不能错的场景（如电商交易交易中的库存和订单、优惠券），可以回答使用成熟的如中间件 Seata AT 模式。

```mipsasm
 阿里开源了分布式事务框架seata经历过阿里生产环境大量考验的框架。  seata支持Dubbo，Spring Cloud。
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210711130223963.png)

是 Seata AT 模式，保障强一致性，支持跨多个库修改数据；

- 订单库：增加订单
- 商品库：扣减库存
- 优惠券库：预扣优惠券

### （2）弱一致性场景

对于数据一致性要求没有那些特别严格、或者由不同系统执行子事务的场景，可以回答使用 Seata TCC 保障弱一致性方案

准备好例子：一个不是严格对数据一致性要求、或者由不同系统执行子事务的场景，如电商订单支付服务，更新订单状态，发送成功支付成功消息，只需要保障弱一致性即可。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210711130524193.png)

Seata TCC 模式，保障弱一致性，支持跨多个服务和系统修改数据，在上面的场景中，使用 Seata TCC 模式事务

- 订单服务：修改订单状态
- 通知服务：发送支付状态

### （3）最终一致性场景

基于可靠消息的最终一致性，各个子事务可以较长时间内异步，但数据绝对不能丢的场景。可以使用**异步确保型事务**事。

可以使用基于 MQ 的异步确保型事务，比如电商平台的通知支付结果：

- 积分服务：增加积分
- 会计服务：生成会计记录

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210711130854311.png)
