JUC 框架的基础是 CAS 和自旋，而 CAS 则是利用 Unsafe 类提供的 CAS 操作，而原子类则依靠于 CAS 和自旋。下面几篇文章从源码分析 JUC 框架内的几个重要的原子类。

- [JUC AtomicInteger 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106035343)
- [JUC AtomicIntegerArray 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106041061)
- [JUC AtomicStampedReference 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106037644)

ThreadLocal 本身不在 JUC 框架之中，但它却是一种防止多线程竞争的重要手段。

- [JUC ThreadLocal 源码行级解析 JDK8](https://blog.csdn.net/anlian523/article/details/105523826)
- [听说你看过 ThreadLocal 源码，来面试下这几个问题](https://blog.csdn.net/anlian523/article/details/105624485)

AQS（AbstractQueuedSynchronizer）向下依赖了 CAS 和自旋，向上则提供了一个同步队列的实现，许多 JUC 框架内中的类都直接使用了 AQS 作为内部类。下面几篇文章将从 AQS 提供的几种功能进行深度分析。

- [AQS 深入理解系列（一） 独占锁的获取过程](https://blog.csdn.net/anlian523/article/details/106344926)
- [AQS 深入理解系列（二） 独占锁的释放过程](https://blog.csdn.net/anlian523/article/details/106515311)
- [AQS 深入理解系列（三）共享锁的获取与释放](https://blog.csdn.net/anlian523/article/details/106598739)
- [AQS 深入理解系列（四）Condition 接口的实现](https://blog.csdn.net/anlian523/article/details/106653034)

AQS 中有些函数的具体实现细节，并不是很容易让人理解，这些地方一般都是因为考虑了同步队列变化中的中间状态。

- [AQS 深入理解 hasQueuedPredecessors 源码分析 JDK8](https://blog.csdn.net/anlian523/article/details/106173860)
- [AQS 深入理解 setHeadAndPropagate 源码分析 JDK8](https://blog.csdn.net/anlian523/article/details/106319294)
- [AQS 深入理解 doReleaseShared 源码分析 JDK8](https://blog.csdn.net/anlian523/article/details/106319538)
- [AQS 深入理解 shouldParkAfterFailedAcquire 源码分析 状态为 0 或 PROPAGATE 的情况分析](https://blog.csdn.net/anlian523/article/details/106448512)
- [interrupt()中断对 LockSupport.park()的影响](https://blog.csdn.net/anlian523/article/details/106752414)

JUC 框架中有些同步构件依赖了 AQS 作为实现底层，我们一般使用它们来做到多线程之间的协作。

- [JUC 框架 CountDownLatch 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106820371)
- [JUC 框架 CyclicBarrier 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106881158)
- [JUC 框架 Semaphore 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106933731)
- [JUC 框架 ReentrantReadWriteLock 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106955678)
- [ReentrantReadWriteLock 深入理解读锁的非公平实现](https://blog.csdn.net/anlian523/article/details/106964711)

JUC 框架中也提供了各种用途的集合类。

- [JUC 集合类 CopyOnWriteArrayList 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106978859)
- [JUC 集合类 CopyOnWriteArraySet 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/106984506)
- [JUC 集合类 ConcurrentSkipListMap 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107123092)
- [JUC 集合类 ConcurrentHashMap 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107272724)
- [JUC 集合类 ConcurrentLinkedQueue 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107453433)
- [JUC 集合类 ConcurrentLinkedDeque 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107521407)
- [JUC 集合类 ArrayBlockingQueue 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107577452)
- [JUC 集合类 LinkedBlockingQueue 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107601481)
- [JUC 集合类 LinkedBlockingDeque 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107685245)
- [JUC 集合类 PriorityBlockingQueue 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107703623)
- [JUC 集合类 DelayQueue 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107801405)
- [JUC 集合类 LinkedTransferQueue 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107876299)
- [JUC 集合类 SynchronousQueue 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/107872233)

最后部分将讲解线程池相关源码。

- [JUC 框架 从 Runnable 到 Callable 到 FutureTask 使用浅析](https://blog.csdn.net/anlian523/article/details/108022167)
- [JUC 框架 FutureTask 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/108029564)
- [JUC 框架 CompletableFuture 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/108023786)
- [JUC 线程池 ThreadPoolExecutor 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/108249789)
- [JUC 线程池 ScheduledThreadPoolExecutor 源码解析 JDK8](https://blog.csdn.net/anlian523/article/details/108332058)
