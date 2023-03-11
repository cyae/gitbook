#并发

## 1. disruptor 是什么？

Disruptor 是英国外汇交易公司 LMAX 开发的一个高性能队列，研发的初衷是解决内存队列的延迟问题（在性能测试中发现竟然与 I/O 操作处于同样的数量级）。

基于 Disruptor 开发的系统单线程能支撑每秒 600 万订单，2010 年在 QCon 演讲后，获得了业界关注。

2011 年，企业应用软件专家 Martin Fowler 专门撰写长文介绍 Disruptor。同年 Disruptor 还获得了 Oracle 官方的 Duke 大奖。

目前，包括 Apache Storm、Camel、Log4j 2 在内的很多知名项目都应用了 Disruptor 以获取高性能。

要深入了解 disruptor ，咱们从 Java 的 内置队列开始介绍起。

## 2. Java 内置队列的问题

介绍 Disruptor 之前，我们先来看一看常用的线程安全的内置队列有什么问题。

Java 的内置队列如下表所示。
队列|有界性|锁|数据结构
---|---|---|---
ArrayBlockingQueue|bounded|加锁|arraylist
LinkedBlockingQueue|optionally-bounded|加锁|linkedlist
ConcurrentLinkedQueue|unbounded|无锁|linkedlist
LinkedTransferQueue|unbounded|无锁|linkedlist
PriorityBlockingQueue|unbounded|加锁|heap
DelayQueue|unbounded|加锁|heap

队列的底层一般分成三种：数组、链表和堆。

其中，堆一般情况下是为了实现带有优先级特性的队列，暂且不考虑。

从数组和链表两种数据结构来看，两类结构如下：

- 基于数组线程安全的队列，比较典型的是 ArrayBlockingQueue，它主要通过加锁的方式来保证线程安全；
- 基于链表的线程安全队列分成 LinkedBlockingQueue 和 ConcurrentLinkedQueue 两大类，前者也通过锁的方式来实现线程安全，而后者通过原子变量 compare and swap（以下简称“CAS”）这种**无锁方式**来实现的。

和 ConcurrentLinkedQueue 一样，上面表格中的 LinkedTransferQueue 都是通过原子变量 CAS 这种不加锁的方式来实现的

但是，对 volatile 类型的变量进行 CAS 操作，存在==伪共享==问题

现实编程过程中，加锁通常会严重地影响性能。线程会因为竞争不到锁而被挂起，等锁被释放的时候，线程又会被恢复，这个过程中存在着很大的开销，并且通常会有较长时间的中断，因为当一个线程正在等待锁时，它不能做任何其他事情。如果一个线程在持有锁的情况下被延迟执行，例如发生了缺页错误、调度延迟或者其它类似情况，那么所有需要这个锁的线程都无法执行下去。如果被阻塞线程的优先级较高，而持有锁的线程优先级较低，就会发生优先级反转。

Disruptor 论文中讲述了一个实验：

这个测试程序调用了一个函数，该函数会对一个 64 位的计数器循环自增 5 亿次。 机器环境：2.4G 6 核 运算： 64 位的计数器累加 5 亿次
Method |Time (ms)
---|---
Single thread| 300
Single thread with CAS| 5,700
Single thread with lock| 10,000
Single thread with volatile write| 4,700
Two threads with CAS| 30,000
Two threads with lock| 224,000

CAS 操作比单线程无锁慢了 1 个数量级；有锁且多线程并发的情况下，速度比单线程无锁慢 3 个数量级。可见无锁速度最快。 **单线程情况下，不加锁的性能 > CAS 操作的性能 > 加锁的性能**。 在多线程情况下，为了保证线程安全，必须使用 CAS 或锁，这种情况下，CAS 的性能超过锁的性能，前者大约是后者的 8 倍。

可以和 BlockingQueue 做对比，不过 disruptor 除了能完成同样的工作场景外，能做更多的事，效率也更高。业务逻辑处理器完全是运行在内存中(in-memory)，使用事件源驱动方式(event sourcing). 业务逻辑处理器的核心是**Disruptors，这是一个并发组件，能够在无锁的情况下实现 Queue 并发安全操作**。

## 3. Disruptor 框架结构

![[Pasted image 20230311232721.png]]

### ringbuffer

数据缓冲区，不同线程之间传递数据的 BUFFER。RingBuffer 是存储消息的地方，通过一个名为 cursor 的 Sequence 对象指示队列的头，协调多个生产者向 RingBuffer 中添加消息，并用于在消费者端判断 RingBuffer 是否为空。

巧妙的是，表示队列尾的 Sequence 并没有在 RingBuffer 中，而是由消费者维护。这样的好处是多个消费者处理消息的方式更加灵活，可以在一个 RingBuffer 上实现消息的单播，多播，流水线以及它们的组合。在 RingBuffer 中维护了一个名为 gatingSequences 的 Sequence 数组来跟踪相关 Seqence。

#### RingBuffer 是什么

RingBuffer 是一个环(首尾相连的环)，用做在不同上下文(线程)间传递数据的 buffer。

RingBuffer 拥有一个序号，这个序号指向数组中下一个可用元素。

![img](https://img-blog.csdnimg.cn/20191204153302150.png)

#### Disruptor 使用环形队列的优势：

Disruptor 框架就是一个使用 CAS 操作的内存队列，与普通的队列不同，

Disruptor 框架使用的是一个基于数组实现的环形队列，无论是生产者向缓冲区里提交任务，还是消费者从缓冲区里获取任务执行，都使用 CAS 操作。

使用环形队列的优势：

第一，简化了多线程同步的复杂度。

学数据结构的时候，实现队列都要两个指针 head 和 tail 来分别指向队列的头和尾，对于一般的队列是这样，

想象下，如果有多个生产者同时往缓冲区队列中提交任务，某一生产者提交新任务后，tail 指针都要做修改的，那么多个生产者提交任务，头指针不会做修改，但会对 tail 指针产生冲突，

例如某一生产者 P1 要做写入操作，在获得 tail 指针指向的对象值 V 后，执行 compareAndSet（）方法前，tail 指针被另一生产者 P2 修改了，这时生产者 P1 执行 compareAndSet（）方法，发现 tail 指针指向的值 V 和期望值 E 不同，导致冲突。

同样，如果多个消费者不断从缓冲区中获取任务，不会修改尾指针，但会造成队列头指针 head 的冲突问题（因为队列的 FIFO 特点，出列会从头指针出开始）。

环形队列的一个特点就是只有一个指针，只通过一个指针来实现出列和入列操作。

如果使用两个指针 head 和 tail 来管理这个队列，有可能会出现“伪共享”问题（伪共享问题在下面我会详细说），

因为创建队列时，head 和 tail 指针变量常常在同一个缓存行中，多线程修改同一缓存行中的变量就容易出现伪共享问题。

第二，由于使用的是环形队列，那么队列创建时大小就被固定了，

Disruptor 框架中的环形队列本来也就是基于数组实现的，使用数组的话，减少了系统对内存空间管理的压力，

因为数组不像链表，Java 会定期回收链表中一些不再引用的对象，而数组不会出现空间的新分配和回收问题。

### Producer/Consumer

Producer 即生产者，比如下图中的 P1. 只是泛指调用 Disruptor 发布事件(我们把写入缓冲队列的一个元素定义为一个事件)的用户代码。

Consumer 和 EventProcessor 是一个概念，新的版本中由 EventProcessor 概念替代了 Consumer。

有两种实现策略，一个是 SingleThreadedStrategy（单线程策略）另一个是 MultiThreadedStrategy（多线程策略），两种策略对应的实现类为 SingleProducerSequencer、MultiProducerSequencer，都实现了 Sequencer 类，之所以叫 Sequencer 是因为他们都是通过 Sequence 来实现数据写，它们定义在生产者和消费者之间快速、正确地传递数据的并发算法。

具体使用哪个根据自己的场景来定，_多线程的策略使用了 AtomicLong（Java 提供的 CAS 操作），而单线程的使用 long，没有锁也没有 CAS。这意味着单线程版本会非常快，因为它只有一个生产者，不会产生序号上的冲突_

Producer 生产 event 数据，EventHandler 作为消费者消费 event 并进行逻辑处理。消费消息的进度通过 Sequence 来控制。

### Sequence 的结构和源码

在 Disruptor 中有一个重要的类 Sequence，该类包装了一个 volatile 修饰的 long 类型数据 value，是一个递增的序列号，通过顺序递增的序号来编号管理通过其进行交换的数据（事件），对数据(事件)的处理过程总是沿着序号逐个递增处理。一个 Sequence 用于跟踪标识某个特定的事件处理者( RingBuffer/Consumer )的处理进度。生产者对 RingBuffer 的互斥访问，生产者与消费者之间的协调以及消费者之间的协调，都是通过 Sequence 实现。几乎每一个重要的组件都包含 Sequence。

无论是 Disruptor 中的基于数组实现的缓冲区 RingBuffer，还是生产者，消费者，都有各自独立的 Sequence。

Sequence 的用途是啥呢？

- 在 RingBuffer 缓冲区中，Sequence 标示着写入进度，例如每次生产者要写入数据进缓冲区时，都要调用 RingBuffer.next（）来获得下一个可使用的相对位置。
- 对于生产者和消费者来说，Sequence 标示着它们的事件序号。

Sequence 的结构图如下

![在这里插入图片描述](https://img-blog.csdnimg.cn/6ec91a2aaeb2421886aead2c3d6133c1.png)

来看看 Sequence 类的源码：

```scala
class LhsPadding {
    protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding {
    protected volatile long value;
}

class RhsPadding extends Value {
    protected long p9, p10, p11, p12, p13, p14, p15;
}

public class Sequence extends RhsPadding {
    static final long INITIAL_VALUE = -1L;
    private static final Unsafe UNSAFE;
    private static final long VALUE_OFFSET;
    static {
        UNSAFE = Util.getUnsafe();
        try {
            VALUE_OFFSET = UNSAFE.objectFieldOffset(Value.class.getDeclaredField("value"));
        } catch(final Exception e) {
             throw new RuntimeException(e);
        }
    }


    public Sequence() {
        this(INITIAL_VALUE);
    }

    public Sequence(final long initialValue) {
        UNSAFE.putOrderedLong(this, VALUE_OFFSET, initialValue);
    }

}
```

### Sequence Barrier

用于保持对 RingBuffer 的 main published Sequence 和 Consumer**依赖**的其它 Consumer 的 Sequence 的引用。 Sequence Barrier 还定义了决定 Consumer 是否还有可处理的事件的逻辑。

SequenceBarrier 用来在消费者之间以及消费者和 RingBuffer 之间**建立依赖关系**。在 Disruptor 中，依赖关系实际上指的是 Sequence 的大小关系，消费者 A 依赖于消费者 B 指的是消费者 A 的 Sequence 一定要小于等于消费者 B 的 Sequence，这种大小关系决定了处理某个消息的先后顺序。因为所有消费者都依赖于 RingBuffer，所以消费者的 Sequence 一定小于等于 RingBuffer 中名为 cursor 的 Sequence，即消息一定是先被生产者放到 Ringbuffer 中，然后才能被消费者处理。

SequenceBarrier 在初始化的时候会收集需要依赖的组件的 Sequence，RingBuffer 的 cursor 会被自动的加入其中。需要依赖其他消费者和/或 RingBuffer 的消费者在消费下一个消息时，会先等待在 SequenceBarrier 上，直到所有被依赖的消费者和 RingBuffer 的 Sequence 大于等于这个消费者的 Sequence。当被依赖的消费者或 RingBuffer 的 Sequence 有变化时，会通知 SequenceBarrier 唤醒等待在它上面的消费者。

### Wait Strategy

当消费者等待在 SequenceBarrier 上时，有许多可选的等待策略，不同的等待策略在延迟和 CPU 资源的占用上有所不同，可以视应用场景选择：

- BusySpinWaitStrategy ： 自旋等待，类似 Linux Kernel 使用的自旋锁。低延迟但同时对 CPU 资源的占用也多。

- BlockingWaitStrategy ： 使用锁和条件变量。CPU 资源的占用少，延迟大。

- SleepingWaitStrategy ： 在多次循环尝试不成功后，选择让出 CPU，等待下次调度，多次调度后仍不成功，尝试前睡眠一个纳秒级别的时间再尝试。这种策略平衡了延迟和 CPU 资源占用，但延迟不均匀。

- YieldingWaitStrategy ： 是一种充分压榨 CPU 的策略，使用 自旋+yield 的方式来提高性能。 当消费线程（Event Handler threads）的数量小于 CPU 核心数时推荐使用该策略。（多个消费者且大于 CPU 核数可能导致 CPU 接近 100%，需要谨慎使用）

- PhasedBackoffWaitStrategy ： 上面多种策略的综合，CPU 资源的占用少，延迟大。

### Event

在 Disruptor 的语义中，生产者和消费者之间进行交换的数据被称为事件(Event)。它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者定义并指定。

#### EventProcessor

EventProcessor 持有特定消费者(Consumer)的 Sequence，并提供用于调用事件处理实现的事件循环(Event Loop)。通过把 EventProcessor 提交到线程池来真正执行，有两类 Processor:

其中一类消费者是 BatchEvenProcessor。每个 BatchEvenProcessor 有一个 Sequence，来记录自己消费 RingBuffer 中消息的情况。所以，一个消息必然会被每一个 BatchEvenProcessor 消费。

另一类消费者是 WorkProcessor。每个 WorkProcessor 也有一个 Sequence，多个 WorkProcessor 还共享一个 Sequence 用于互斥的访问 RingBuffer。一个消息被一个 WorkProcessor 消费，就不会被共享一个 Sequence 的其他 WorkProcessor 消费。这个被 WorkProcessor 共享的 Sequence 相当于尾指针

#### EventHandler

Disruptor 定义的事件处理接口，由用户实现，用于处理事件，是 Consumer 的真正实现。开发者实现 EventHandler，然后作为入参传递给 EventProcessor 的实例。

## 4. Disruptor 的使用场景

Disruptor 它可以用来作为高性能的有界内存队列， 适用于两大场景：

- 生产者消费者场景
- 发布订阅 场景

生产者消费者场景。Disruptor 的最常用的场景就是“生产者-消费者”场景，对场景的就是“一个生产者、多个消费者”的场景，并且要求顺序处理。

> 备注，这里和 JCTool 的 MPSC 队列，刚好相反， MPSC 使用于多生产者，单消费者场景

发布订阅 场景：Disruptor 也可以认为是观察者模式的一种实现， 实现发布订阅模式。

当前业界开源组件使用 Disruptor 的包括 Log4j2、Apache Storm 等，

### 实战 1：Disruptor 的 使用实例

我们从一个简单的例子开始学习 Disruptor：

生产者传递一个 long 类型的值给消费者，而消费者消费这个数据的方式仅仅是把它打印出来。

#### 定义一个 Event 和工厂

首先定义一个 Event 来包含需要传递的数据：

```csharp
public class LongEvent {
    private long value;
    public long getValue() {
        return value;
    }

    public void setValue(long value) {
        this.value = value;
    }
}
```

由于需要让 Disruptor 为我们创建事件，我们同时还声明了一个 EventFactory 来创建 Event 对象。

```typescript
public class LongEventFactory implements EventFactory {
    @Override
    public Object newInstance() {
        return new LongEvent();
    }
}
```

#### 定义事件处理器（消费者）

我们还需要一个事件消费者，也就是一个事件处理器。

这个例子中，事件处理器的工作，就是简单地把事件中存储的数据打印到终端：

```java
    /**
     * 类似于消费者
     *  disruptor会回调此处理器的方法
     */
    static class LongEventHandler implements EventHandler<LongEvent> {
        @Override
        public void onEvent(LongEvent longEvent, long l, boolean b) throws Exception {
            System.out.println(longEvent.getValue());
        }
    }
```

disruptor 会回调此处理器的方法

#### 定义事件源(生产者)

事件都会有一个生成事件的源，类似于 生产者的角色，

如何产生事件，然后发出事件呢？

通过从 环形队列中 获取 序号， 通过序号获取 对应的 事件对象， 将数据填充到 事件对象，再通过 序号将 事件对象 发布出去。

一段生产者的代码如下：

```csharp


    //  事件生产者：业务代码
    // 通过从 环形队列中 获取 序号， 通过序号获取 对应的 事件对象， 将数据填充到 事件对象，再通过 序号将 事件对象 发布出去。
    static class LongEventProducer {
        private final RingBuffer<LongEvent> ringBuffer;

        public LongEventProducer(RingBuffer<LongEvent> ringBuffer) {
            this.ringBuffer = ringBuffer;
        }

        /**
         * onData用来发布事件，每调用一次就发布一次事件事件
         * 它的参数会通过事件传递给消费者
         *
         * @param data
         */
        public void onData(long data) {

            // step1：通过从 环形队列中 获取 序号
            //可以把ringBuffer看做一个事件队列，那么next就是得到下面一个事件槽
            long sequence = ringBuffer.next();

            try {

                //step2: 通过序号获取 对应的 事件对象， 将数据填充到 事件对象，
                //用上面的索引，取出一个空的事件用于填充
                LongEvent event = ringBuffer.get(sequence);// for the sequence
                event.setValue(data);
            } finally {

                //step3: 再通过 序号将 事件对象 发布出去。
                //发布事件
                ringBuffer.publish(sequence);
            }
        }
    }

```

很明显的是：

当用一个简单队列来发布事件的时候会牵涉更多的细节，这是因为事件对象还需要预先创建。

发布事件最少需要三步：

step1：获取下一个事件槽。

如果我们使用 RingBuffer.next()获取一个事件槽，那么一定要发布对应的事件。

step2: 通过序号获取 对应的 事件对象， 将数据填充到 事件对象，

step3: 再通过 序号将 事件对象 发布出去。

发布事件的时候要使用 try/finnally 保证事件一定会被发布

如果不能发布事件，那么就会引起 Disruptor 状态的混乱。

尤其是在多个事件生产者的情况下会导致事件消费者失速，从而不得不重启应用才能会恢复。

Disruptor 3.0 提供了 lambda 式的 API。

这样可以把一些复杂的操作放在 Ring Buffer，所以在 Disruptor3.0 以后的版本最好使用 Event Publisher 或者 Event Translator(事件转换器)来发布事件。

#### 组装起来

最后一步就是把所有的代码组合起来完成一个完整的事件处理系统。

```java
  @org.junit.Test
    public  void testSimpleDisruptor() throws InterruptedException {
        // 消费者线程池
        Executor executor = Executors.newCachedThreadPool();
        // 事件工厂
        LongEventFactory eventFactory = new LongEventFactory();
        // 环形队列大小，2的指数
        int bufferSize = 1024;

        // 构造  分裂者 （事件分发者）
        Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(eventFactory, bufferSize, executor);

        // 连接 消费者 处理器
        disruptor.handleEventsWith(new LongEventHandler());
        // 开启 分裂者（事件分发）
        disruptor.start();

        // 获取环形队列，用于生产 事件
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        LongEventProducer producer = new LongEventProducer(ringBuffer);

        for (long i = 0; true; i++) {
            //发布事件
            producer.onData(i);
            Thread.sleep(1000);
        }
    }
```

#### 事件转换器

Disruptor3.0 以后 , 提供了事件转换器， 帮助填充 LongEvent 的业务数据

下面是一个例子

```csharp
  static class LongEventProducerWithTranslator {
        //一个translator可以看做一个事件初始化器，publicEvent方法会调用它
        //填充Event
        private static final EventTranslatorOneArg<LongEvent, Long> TRANSLATOR =
                new EventTranslatorOneArg<LongEvent, Long>() {
                    public void translateTo(LongEvent event, long sequence, Long data) {
                        event.setValue(data);
                    }
                };

        private final RingBuffer<LongEvent> ringBuffer;

        public LongEventProducerWithTranslator(RingBuffer<LongEvent> ringBuffer) {
            this.ringBuffer = ringBuffer;
        }

        public void onData(Long data) {
            ringBuffer.publishEvent(TRANSLATOR, data);
        }
    }
```

使用事件转换器的好处，省了从 环形队列 获取 序号， 然后拿到事件 填充数据， 再发布序号 中的第二步骤

给 事件 填充 数据 的动作，在 EventTranslatorOneArg 完成

Disruptor 提供了不同的接口去产生一个 Translator 对象：

- EventTranslator,
- EventTranslatorOneArg,
- EventTranslatorTwoArg,

很明显，Translator 中方法的参数是通过 RingBuffer 来传递的。

使用 事件转换器 转换器的进行事件的 生产与消费 代码，大致如下：

```java
   @org.junit.Test
    public void testSimpleDisruptorWithTranslator() throws InterruptedException {
        // 消费者线程池
        Executor executor = Executors.newCachedThreadPool();
        // 事件工厂
        LongEventFactory eventFactory = new LongEventFactory();
        // 环形队列大小，2的指数
        int bufferSize = 1024;

        // 构造  分裂者 （事件分发者）
        Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(eventFactory, bufferSize, executor);

        // 连接 消费者 处理器
        disruptor.handleEventsWith(new LongEventHandler());
        // 开启 分裂者（事件分发）
        disruptor.start();

        // 获取环形队列，用于生产 事件
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);

        for (long i = 0; true; i++) {
            //发布事件
            producer.onData(i);
            Thread.sleep(1000);
        }
    }
```

上面写法的另一个好处是，Translator 可以分离出来并且更加容易单元测试。

#### 通过 Java 8 Lambda 使用 Disruptor

Disruptor 在自己的接口里面添加了对于 Java 8 Lambda 的支持。

大部分 Disruptor 中的接口都符合 Functional Interface 的要求（也就是在接口中仅仅有一个方法）。

所以在 Disruptor 中，可以广泛使用 Lambda 来代替自定义类。

```java
 @org.junit.Test
    public void testSimpleDisruptorWithLambda() throws InterruptedException {
        // 消费者线程池
        Executor executor = Executors.newCachedThreadPool();
        // 环形队列大小，2的指数
        int bufferSize = 1024;

        // 构造  分裂者 （事件分发者）
        Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(LongEvent::new, bufferSize, executor);

        // 连接 消费者 处理器
        // 可以使用lambda来注册一个EventHandler
        disruptor.handleEventsWith((event, sequence, endOfBatch) -> System.out.println("Event: " + event.getValue()));
        // 开启 分裂者（事件分发）
        disruptor.start();

        // 获取环形队列，用于生产 事件
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);

        for (long i = 0; true; i++) {
            //发布事件
            producer.onData(i);
            Thread.sleep(1000);
        }
    }
```

由于在 Java 8 中方法引用也是一个 lambda，因此还可以把上面的代码改成下面的代码：

```java

    public static void handleEvent(LongEvent event, long sequence, boolean endOfBatch)
    {
        System.out.println(event.getValue());
    }

    @org.junit.Test
    public void testSimpleDisruptorWithMethodRef() throws InterruptedException {
        // 消费者线程池
        Executor executor = Executors.newCachedThreadPool();
        // 环形队列大小，2的指数
        int bufferSize = 1024;

        // 构造  分裂者 （事件分发者）
        Disruptor<LongEvent> disruptor = new Disruptor<LongEvent>(LongEvent::new, bufferSize, executor);

        // 连接 消费者 处理器
        // 可以使用lambda来注册一个EventHandler
        disruptor.handleEventsWith(LongEventDemo::handleEvent);
        // 开启 分裂者（事件分发）
        disruptor.start();

        // 获取环形队列，用于生产 事件
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();

        LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);

        for (long i = 0; true; i++) {
            //发布事件
            producer.onData(i);
            Thread.sleep(1000);
        }
    }
}
```

### 实战 2：停车批量数据上报

停车批量入场数据上报，数据上报后需要对每条入场数据存入 DB，还需要发送 kafka 消息给其他业务系统。如果执行完所有的操作，再返回，那么接口耗时比较长，我们可以批量上报后验证数据正确性，通过后按单条入场数据写入环形队列，然后直接返回成功。

实现方式一：启 动 2 个消费者线程，一个消费者去执行 db 入库，一个消费者去发送 kafka 消息。

实现方式二：启动 4 个消费者，2 个消费者并发执行 db 入库，两个消费者并发发送 kafka 消息，充分利用 cpu 多核特性，提高执行效率。

实现方式三：如果要求写入 DB 和 kafka 后，需要给用户发送短信。那么可以启动三个消费者线程，一个执行 db 插入，一个执行 kafka 消息发布，最后一个依赖前两个线程执行成功，前两个线程都执行成功后，该线程执行短信发送。

```java
/**
 * 测试 P1生产消息，C1，C2消费消息，C1和C2会共享所有的event元素! C3依赖C1，C2处理结果
 *
 */
public class Main {

    public static void main(String[] args) throws InterruptedException {
        long beginTime = System.currentTimeMillis();

        //最好是2的n次方
        int bufferSize = 1024;
        // Disruptor交给线程池来处理，共计 p1,c1,c2,c3四个线程
        ExecutorService executor = Executors.newFixedThreadPool(4);
        // 构造缓冲区与事件生成
        Disruptor<InParkingDataEvent> disruptor = new Disruptor<InParkingDataEvent>(
                new EventFactory<InParkingDataEvent>() {
                    @Override
                    public InParkingDataEvent newInstance() {
                        return new InParkingDataEvent();
                    }
                }, bufferSize, executor, ProducerType.SINGLE, new YieldingWaitStrategy());

        // 使用disruptor创建消费者组C1,C2
        EventHandlerGroup<InParkingDataEvent> handlerGroup = disruptor.handleEventsWith(new ParkingDataToKafkaHandler(),
                new ParkingDataInDbHandler());

        ParkingDataSmsHandler smsHandler = new ParkingDataSmsHandler();
        // 声明在C1,C2完事之后执行JMS消息发送操作 也就是流程走到C3
        handlerGroup.then(smsHandler);

        disruptor.start();// 启动
        CountDownLatch latch = new CountDownLatch(1);
        // 生产者准备
        executor.submit(new InParkingDataEventPublisher(latch, disruptor));
        latch.await();// 等待生产者结束
        disruptor.shutdown();
        executor.shutdown();

        System.out.println("总耗时:" + (System.currentTimeMillis() - beginTime));
    }
}
```

### 实战 3：订单系统

你在网上使用信用卡下订单。一个简单的零售系统将获取您的订单信息，使用信用卡验证服务，以检查您的信用卡号码，然后确认您的订单。

所有这些都在一个单一过程中操作。当进行信用卡有效性检查时，服务器这边的线程会阻塞等待，当然这个对于用户来说停顿不会太长。

在 MAX 架构中，你将此单一操作过程分为两个，第一部分将获取订单信息，然后输出事件(请求信用卡检查有效性的请求事件)给信用卡公司. 业务逻辑处理器将继续处理其他客户的订单，直至它在输入事件中发现了信用卡已经检查有效的事件，然后获取该事件来确认该订单有效。

### 实战 4：文件 hash

文件中存放 50 亿个 url，每个 url 各占 64 字节，内存限制是 4G。按照每个 url64 字节来算，每个文件有 50 亿个 url，那么每个文件大小为 5G\*64=320G。320G 远远超出内存限定的 4G，分给四个线程分别处理，每个线程处理 80G 文件内容，单线程内需要循环处理 20 次。每次处理完后，根据 url 的 hash 值输出到对应小文件，然后进行下一次处理。

![[Pasted image 20230311235138.png]]

这样的方法有两个弊端：

- 同一个线程内，读写相互依赖，互相等待
- 不同线程可能争夺同一个输出文件，需要 lock 同步

于是改为如下方法，四个线程读取数据，计算 hash 值，将信息写入相应 disruptor。每个线程对应 disruptor 的一个消费者，将 disruptor 中的信息落盘持久化（使用 disruptor 的多生产者单消费者模型）。对于四个读取线程(生产者)而言，只有读取文件操作，没有写文件操作，因此不存在读写互相依赖的问题。对于 disruptor 消费线程而言，只存在写文件操作，没有读文件，因此也不存在读写互相依赖的问题。同时 disruptor 的单消费者又很好的解决了多个线程互相竞争同一个文件的问题(disruptor 的一个消费者是相当于一个线程)，因此可以大大提高程序的吞吐率。

![[Pasted image 20230311235230.png]]

```jaVA
/**
 * 多生产者多消费者模型
 * @author lzhcode
 *
 */
public class Main {

    public static void main(String[] args) throws InterruptedException {

        //1 创建RingBuffer
        RingBuffer<Order> ringBuffer =
                RingBuffer.create(ProducerType.MULTI,
                        new EventFactory<Order>() {
                            public Order newInstance() {
                                return new Order();
                            }
                        },
                        1024*1024,
                        new YieldingWaitStrategy());

        //2 通过ringBuffer 创建一个屏障
        SequenceBarrier sequenceBarrier = ringBuffer.newBarrier();

        //3 创建一个消费者:
        Consumer consumer = new Consumer;


        //4 构建多消费者工作池
        WorkerPool<Order> workerPool = new WorkerPool<Order>(
                ringBuffer,
                sequenceBarrier,
                new EventExceptionHandler(),
                consumer );

        //5 设置消费者的sequence序号 用于单独统计消费进度, 并且设置到ringbuffer中
        ringBuffer.addGatingSequences(workerPool.getWorkerSequences());

        //6 启动workerPool
        workerPool
        .start(Executors.newFixedThreadPool(5));

        final CountDownLatch latch = new CountDownLatch(1);

        for(int i = 0; i < 100; i++) {
            final Producer producer = new Producer(ringBuffer);
            new Thread(new Runnable() {
                public void run() {
                    try {
                        latch.await();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                    for(int j = 0; j<100; j++) {
                        producer.sendData(UUID.randomUUID().toString());
                    }
                }
            }).start();
        }

        Thread.sleep(2000);
        System.err.println("----------线程创建完毕，开始生产数据----------");
        latch.countDown();

        Thread.sleep(10000);

        System.err.println("任务总数:" + consumers[2].getCount());
    }

    static class EventExceptionHandler implements ExceptionHandler<Order> {
        public void handleEventException(Throwable ex, long sequence, Order event) {
        }

        public void handleOnStartException(Throwable ex) {
        }

        public void handleOnShutdownException(Throwable ex) {
        }
    }

}
```

### 实战 5：Netty 整合 Disruptor 实现百万长连接服务架构

对于一个 server，我们一般考虑他所能支撑的 qps，但有那么一种应用， 我们需要关注的是它能支撑的连接数个数，而并非 qps，当然 qps 也是我们需要考虑的性能点之一。这种应用常见于消息推送系统，比如聊天室或即时消息推送系统等。c 对于这类系统，因为很多消息需要到产生时才推送给客户端，所以当没有消 息产生时，就需要 hold 住客户端的连接，这样，当有大量的客户端时，就需要 hold 住大量的连接，这种连接我们称为长连接。

首先，我们分析一下，对于这类服务，需消耗的系统资源有：cpu、网络、内存。所以，想让系统性能达到最佳，我们先找到系统的瓶颈所在。这样的长连 接，往往我们是没有数据发送的，所以也可以看作为非活动连接。对于系统来说，这种非活动连接，并不占用 cpu 与网络资源，而仅仅占用系统的内存而已。所以，我们假想，只要系统内存足够，系统就能够支持我们想达到的连接数，那么事实是否真的如此？

在 Linux 内核配置上，默认的配置会限制全局最大打开文件数(Max Open Files)还会限制进程数。 所以需要对 Linux 内核配置进行一定的修改才可以。具体如何修改这里不做讨论

java 中用的是非阻塞 IO（NIO 和 AIO 都算），那么它们都可以用单线程来实现大量的 Socket 连接。 不会像 BIO 那样为每个连接创建一个线程，因为代码层面不会成为瓶颈，最主要的是把业务代码用 disruptor 来进行解耦

![[Pasted image 20230311235446.png]]

参考代码：

- https://gitee.com/lzhcode/maven-parent/tree/master/lzh-disruptor/lzh-disruptor-netty-server
- https://gitee.com/lzhcode/maven-parent/tree/master/lzh-disruptor/lzh-disruptor-netty-com
- https://gitee.com/lzhcode/maven-parent/tree/master/lzh-disruptor/lzh-disruptor-netty-client

### 更多实战

[Disruptor 框架中生产者、消费者的各种复杂依赖场景下的使用总结](https://www.cnblogs.com/pku-liuqiang/p/8544700.html)

## 5. 构造 Disruptor 对象的几个要点

在构造 Disruptor 对象，有几个核心的要点：

- 事件工厂(Event Factory)定义了如何实例化事件(Event)，Disruptor 通过 EventFactory 在 RingBuffer 中预创建 Event 的实例。
- ringBuffer 这个数组的大小，一般根据业务指定成 2 的指数倍。
- 消费者线程池，事件的处理是在构造的线程池里来进行处理的。
- 指定等待策略，Disruptor 定义了 com.lmax.disruptor.WaitStrategy 接口用于抽象  **Consumer 如何等待 Event 事件**。

Disruptor 提供了多个 WaitStrategy 的实现，每种策略都具有不同性能和优缺点，根据实际运行环境的 CPU 的硬件特点选择恰当的策略，并配合特定的 JVM 的配置参数，能够实现不同的性能提升。

- BlockingWaitStrategy 是最低效的策略，但其对**CPU 的消耗最小**并且在各种不同部署环境中能提供更加一致的性能表现；
- SleepingWaitStrategy 的性能表现跟 BlockingWaitStrategy 差不多，对 CPU 的消耗也类似，但其对生产者线程的影响最小，适合用于异步日志类似的场景；
- YieldingWaitStrategy 的性能是最好的，适合用于低延迟的系统。在要求极高性能且**事件处理线数小于 CPU 逻辑核心数的场景中**，推荐使用此策略；。

## Disruptor 如何实现高性能？

使用 Disruptor，主要用于对性能要求高、延迟低的场景，它通过“榨干”机器的性能来换取处理的高性能。

Disruptor 实现高性能主要体现了去掉了锁，采用 CAS 算法，同时内部通过环形队列实现有界队列。

- 环形数据结构  
  数组元素不会被回收，避免频繁的 GC，所以，为了避免垃圾回收，采用数组而非链表。

  同时，数组对处理器的缓存机制更加友好，可以开大数组批量预读。

- 元素位置定位  
  数组长度 $2^n$，通过位运算，加快定位的速度。

  下标采取递增的形式。不用担心 index 溢出的问题。

  index 是 long 类型，即使 100 万 QPS 的处理速度，也需要 30 万年才能用完。

- 无锁设计

  对于单生产者，可做到完全无锁。

  对于多生产者，采用 CAS 无锁方式，保证线程的安全性

  每个生产者或者消费者线程，会先申请可以操作的元素在数组中的位置，申请到之后，直接在该位置写入或者读取数据。整个过程通过原子变量 CAS，保证操作的线程安全。

- 属性填充：

  通过添加额外的无用信息，避免伪共享问题

### Disruptor 和 BlockingQueue 比较:

- **BlockingQueue:** FIFO 队列.生产者 Producer 向队列中发布 publish 一个事件时,消费者 Consumer 能够获取到通知.如果队列中没有消费的事件,消费者就会被阻塞,直到生产者发布新的事件
- Disruptor 可以比 BlockingQueue 做到更多:
  - Disruptor 队列中同一个事件可以有多个消费者,消费者之间既可以并行处理,也可以形成依赖图相互依赖,按照先后次序进行处理
  - Disruptor 可以预分配用于存储事件内容的内存空间
  - Disruptor 使用极度优化和无锁的设计实现极高性能的目标

如果你的项目有对性能要求高，对延迟要求低的需求，并且需要一个无锁的有界队列，来实现生产者/消费者模式，那么 Disruptor 是你的不二选择。


## 关闭 Disruptor

- **disruptor.shutdown() :**  关闭Disruptor。方法会阻塞,直至所有的事件都得到处理
- **executor.shutdown() :**  关闭Disruptor使用的线程池。如果线程池需要关闭,必须进行手动关闭 Disruptor在shutdown时不会自动关闭使用的线程池
- [Disruptor 中 shutdown 方法失效，及产生的不确定性源码分析](https://www.cnblogs.com/pku-liuqiang/p/8532737.html)
