通过前面内容我们了解了 synchronized，虽然 JVM 对它做了很多优化，但是它还是一个**重量级**的锁。而接下来要介绍的 volatile 则是**轻量级**的 synchronized。

如果一个变量使用 volatile，则它比使用 synchronized 的成本更加低，因为它**不会引起线程上下文的切换和调度**。

Java 语言规范对 volatile 的定义如下：

Java 允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。

通俗点讲就是说一个变量如果用 volatile 修饰了，则 Java 可以确保所有线程看到这个变量的值是一致的，如果某个线程对 volatile 修饰的共享变量进行更新，那么其他线程可以立马看到这个更新，这就是内存可见性。

volatile 虽然看起来比较简单，使用起来无非就是在一个变量前面加上 volatile 即可，但是要用好并不容易。

## 一、解决内存可见性问题

在可见性问题案例中进行如下修改，添加 volatile 关键词，变成 volatile 变量：

```java
private volatile boolean flag = true;
```

代码如下：

```java
public class Demo1Jmm {
    public static void main(String[] args) throws InterruptedException {
        JmmDemo demo = new JmmDemo();
        Thread t = new Thread(demo);
        t.start();
        Thread.sleep(100);
        demo.flag = false;
        System.out.println("已经修改为false");
        System.out.println(demo.flag);
    }

    static class JmmDemo implements Runnable {
        public volatile boolean flag = true;
        public void run() {
            System.out.println("子线程执行。。。");
            while (flag) {
            }
            System.out.println("子线程结束。。。");
        }
    }
}
```

结果如下：

```bash
子线程执行。。。
已经修改为false
子线程结束。。。
false
```

Volatile 实现内存可见性的过程

线程写 Volatile 变量的过程：

1. 改变线程本地内存中 Volatile 变量副本的值；
2. 将改变后的副本的值从本地内存刷新到主内存；

线程读 Volatile 变量的过程：

1. 从主内存中读取 Volatile 变量的最新值到线程的本地内存中；
2. 从本地内存中读取 Volatile 变量的副本

Volatile 实现内存可见性原理（了解）：

写操作时，通过在写操作指令后加入一条 store 屏障指令，让本地内存中变量的值能够刷新到主内存中；

读操作时，通过在读操作前加入一条 load 屏障指令，及时读取到变量在主内存的值

PS: 内存屏障（Memory Barrier）是一种 CPU 指令，用于控制特定条件下的重排序和内存可见性问题。Java 编译器也会根据内存屏障的规则禁止重排序

volatile 的底层实现是通过插入内存屏障，但是对于编译器来说，发现一个最优布置来最小化插入内存屏障的总数几乎是不可能的，所以，JMM 采用了保守策略。如下：

1. StoreStore 屏障可以保证在 volatile 写之前，其前面的所有普通**写操作都已经刷新到主内存**中。
2. StoreLoad 屏障的作用是**避免 volatile 写与后面可能有的 volatile 读/写操作重排序**。
3. LoadLoad 屏障用来禁止处理器把上面的 volatile 读与下面的普通读重排序。
4. LoadStore 屏障用来禁止处理器把上面的 volatile 读与下面的普通写重排序。

java 内存模型为了实现`volatile`可见性和**禁止指令重排**两个语义，使用如下内存屏障插入策略：

1. 每个`volatile`写操作前边插入`Store-Store`屏障，后边插入`Store-Load(全能)`屏障；
2. 每个`volatile`读操作前边插入`Load-Load`屏障和`Load-Stroe`屏障；

![](https://img2022.cnblogs.com/blog/2138872/202202/2138872-20220207202943624-1853087839.png)

如图所示：写`volatile`屏障指令插入策略可以保证在`volatile`写之前，所有写操作都已经刷新到主存对所有处理器可见了。其后全能型屏障指令为了避免写`volatile`与其后`volatile`读写指令重排序。

![](https://img2022.cnblogs.com/blog/2138872/202202/2138872-20220207203108834-456330018.png)

读`volatile`时，会在其后插入两条指令防止`volatile`读操作与其后的读写操作重排序。

## 二、解决重排问题

> 如果操作 X happens-before 操作 Y，那么 X 的结果对于 Y 可见。

- happens-before 模型包括如下几种:

  - 线程内部字节码顺序, 注意如果操作没有依赖关系, jvm 在编译阶段可能已将字节码重排
  - unlock -> lock
  - volatile 写 -> volatile 读
  - Thread.start() -> run()的首个操作操作
  - run()的末尾操作 -> Thread.isAlive()/Thread.join()
  - Thread.interrupt() -> catch InterruptedException/Thread.interrupted()
  - 构造器的末尾操作 -> finalizer 的首个操作

- 如下示例代码, 由于方法局部变量之间没有依赖性, 所以在多线程情况下的重排可能引发意外结果

```java
int a=0, b=0;

public void method1() {
  int r2 = a;
  b = 1;
}

public void method2() {
  int r1 = b;
  a = 2;
}
```

- 使用 volatile 修饰 b, 强制规定 b 的写操作 先于 b 的读操作, 结果一定为 r1 = 1, r2 = 0;
  - 本质上是利用 happens-before 规定了多线程的运行顺序

```java
int a=0;
volatile int b=0;

public void method1() {
  int r2 = a; // 1. read a
  b = 1; // 2. write b
}

public void method2() {
  int r1 = b; // 3. read b
  a = 2; // 4. write a
}
```

- happens-before 底层实现原理: 内存屏障
- 比如规则: volatile 写 -> volatile 读, 那么 volatile 写读之间的内存访问应该被屏蔽, 即禁止 jvm 将 访问内存字节码 重排到 volatile 写读之间
  - 实现上, 并没有真正禁止重排, 而是遇到 volatile 写就**强制刷新缓存**, 将当前 volatile 值刷回内存, 同时使其他线程持有的同一缓存行失效(类似于先写 DB 后删 redis 缓存的强一致性策略), 保证了即使重排字节码, 也能去内存读到最新值的效果
  - volatile 缺点: 不保证原子性, 如果频繁刷新缓存性能会退化(伪共享问题), 因此只适用于读多写少, 单线程写场景

## 三、原子性的问题

虽然 Volatile 关键字可以让变量在多个线程之间可见，但是 Volatile**不具备**原子性。

```java
public class Demo3Volatile {
  public static void main(String[] args) throws InterruptedException {
    VolatileDemo demo = new VolatileDemo();
    for (int i = 0; i < 5; i++) {
      Thread t = new Thread(demo);
      t.start();
   }
    Thread.sleep(1000);
    System.out.println(demo.count);
 }
  static class VolatileDemo implements Runnable {
    public volatile int count;
    //public volatile AtomicInteger count = new AtomicInteger(0);
    public void run() {
      addCount();
   }
    public void addCount() {
      for (int i = 0; i < 10000; i++) {
        count++;
     }
   }
 }
}
```

结果：48109

以上出现原子性问题的原因是 count++并不是原子性操作。

count = 5 开始，流程分析：

1. 线程 1 读取 count 的值为 5
2. 线程 2 读取 count 的值为 5
3. 线程 2 加 1 操作
4. 线程 2 最新 count 的值为 6
5. 线程 2 写入值到主内存的最新值为 6  
   这个时候，线程 1 的 count 为 5，线程 2 的 count 为 6。如果切换到线程 1 执行，那么线程 1 得到的结果是 6，写入到主内存的值还是 6，现在的情况是对 count 进行了两次加 1 操作，但是主内存实际上只是加 1 一次

解决方案：

- 使用 synchronized
- 使用 ReentrantLock（可重入锁）
- 使用 AtomicInteger（原子操作）

### 使用 synchronized

```java
public class Demo3Volatile {
  public static void main(String[] args) throws InterruptedException {
    VolatileDemo demo = new VolatileDemo();
    for (int i = 0; i < 5; i++) {
      Thread t = new Thread(demo);
      t.start();
   }
    Thread.sleep(1000);
    System.out.println(demo.count);
 }
  static class VolatileDemo implements Runnable {
    public volatile int count;
    //public volatile AtomicInteger count = new AtomicInteger(0);
    public void run() {
      addCount();
   }
    public synchronized void addCount() {
      for (int i = 0; i < 10000; i++) {
        count++;
     }
   }
 }
}
```

结果：50000

### 使用 ReentrantLock（可重入锁）

```java
public class Demo3Volatile {
  public static void main(String[] args) throws InterruptedException {
    VolatileDemo demo = new VolatileDemo();
    for (int i = 0; i < 5; i++) {
      Thread t = new Thread(demo);
      t.start();
   }
    Thread.sleep(1000);
    System.out.println(demo.count);
 }
  static class VolatileDemo implements Runnable {
    private Lock lock = new ReentrantLock();
    public volatile int count;
    //public volatile AtomicInteger count = new AtomicInteger(0);
    public void run() {
      addCount();
   }
    public void addCount() {
      for (int i = 0; i < 10000; i++) {
        lock.lock();
        count++;
        lock.unlock();
     }
   }
 }
}
```

### 使用 AtomicInteger（原子操作）

```java
public class Demo3Volatile {
  public static void main(String[] args) throws InterruptedException {
    VolatileDemo demo = new VolatileDemo();
    for (int i = 0; i < 5; i++) {
      Thread t = new Thread(demo);
      t.start();
   }
    Thread.sleep(1000);
    System.out.println(demo.count);
 }
  static class VolatileDemo implements Runnable {
    public static AtomicInteger count = new AtomicInteger(0);
    public void run() {
      addCount();
   }
    public void addCount() {
      for (int i = 0; i < 10000; i++) {
        count.incrementAndGet();
     }
   }
 }
}
```

结果：50000

## 四、Volatile 适合使用场景

a）对变量的写入操作不依赖其当前值

不满足：number++、count=count\*5 等

满足：boolean 变量、直接赋值的变量等

b）该变量没有包含在具有其他变量的不变式中

不满足：不变式 low\<up

总结：变量真正独立于其他变量和自己以前的值，在单独使用的时候，适合用 volatile

## 五、synchronized 和 volatile 比较

- volatile 不需要加锁，比 synchronized 更轻便，不会阻塞线程
- synchronized 既能保证可见性，又能保证原子性，而 volatile 只能保证可见性，无法保证原子性

与锁相比，Volatile 变量是一种非常简单但同时又非常脆弱的同步机制，它在某些情况下将提供优于锁的性能和伸缩性。如果严格遵循 volatile 的使用条件（变量真正独立于其他变量和自己以前的值 ） 在某些情况下可以使用  volatile  代替  synchronized  来优化代码提升效率。

---

《阿里巴巴开发手册》中：

【参考】 volatile 解决多线程内存不可见问题。

对于一写多读，是可以解决变量同步问题，但是如果多写，同样无法解决线程安全问题。

如果是 count ++操作，使用如下类实现：AtomicInteger count = new AtomicInteger(); count . addAndGet( 1 );

如果是 JDK 8，推荐使用 LongAdder 对象，比 AtomicLong 性能更好 （ 减少乐观锁的重试次数 ）。

## synchronized 锁升级

> 每个对象都拥有字段: markword 对象头, 其末尾 3 位标识该对象的锁状态

- 无锁(001)表示所有线程都能访问
- 偏向锁(101, 默认)针对的是锁仅会被同一线程持有的情况
  - 只会在第一次请求时采用 CAS 操作，在锁对象的 markword 记录下当前线程的地址
  - 即使持有线程已经结束访问, 也不会主动清除标记, 方便这个线程重复加锁时直接放行
  - 如果其他线程申请加锁, 则判断 markword 记录的当前持有线程是否存活
    - 如果持有线程已终止, 则改线程地址为新的申请线程
    - 否则, 表明出现了多线程竞争, 针对单线程的偏向锁已经不能满足, 升级为轻量级锁
- 轻量级锁/自旋锁(00)针对的是多个线程在不同时间段申请同一把锁, 且每个线程持有时间较短的情况
  - 为了减少阻塞操作带来的上下文切换, 竞争线程通过自旋操作保持在用户态, 并不断 CAS 尝试修改标志为 00
  - 这在持有时间较短的情况下是可以接受的(比如本地加锁), 但如果持有时间很长(比如加锁后 RPC), 空转自旋会消耗 CPU, 影响吞吐
  - 因此, 当竞争线程自旋超过一定次数(根据历史自旋次数决定), 或竞争线程很多时, 表示持有时间会较长, 让竞争者都阻塞, 减少 CPU 消耗, 升级为重量级锁
- 重量级锁(10)会阻塞、唤醒请求加锁的线程。它针对的是多个线程同时竞争同一把锁的情况。
  - 类似 threadlocal, 每个对象都拥有字段: 锁计数器, 持有者指针, 阻塞队列
  - 当进入 monitorEnter, 如果计数器=0, 表明可以加锁; 如果计数器>0, 则继续判断锁持有者是否就是访问者(可重入锁), 如果是则可以加锁, 否则进入阻塞队列
  - 当进入 monitorExit, 计数器--, 如果计数器减到 0, 表示锁释放了, 从阻塞队列中取出线程尝试加锁
