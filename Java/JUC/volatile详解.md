
通过前面内容我们了解了synchronized，虽然JVM对它做了很多优化，但是它还是一个**重量级**的锁。而接下来要介绍的volatile则是**轻量级**的synchronized。

如果一个变量使用volatile，则它比使用synchronized的成本更加低，因为它**不会引起线程上下文的切换和调度**。

Java语言规范对volatile的定义如下：

Java允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。

通俗点讲就是说一个变量如果用volatile修饰了，则Java可以确保所有线程看到这个变量的值是一致的，如果某个线程对volatile修饰的共享变量进行更新，那么其他线程可以立马看到这个更新，这就是内存可见性。

volatile虽然看起来比较简单，使用起来无非就是在一个变量前面加上volatile即可，但是要用好并不容易。

## 一、解决内存可见性问题

在可见性问题案例中进行如下修改，添加volatile关键词，变成volatile变量：

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

Volatile实现内存可见性的过程

线程写Volatile变量的过程：

1. 改变线程本地内存中Volatile变量副本的值；
2. 将改变后的副本的值从本地内存刷新到主内存；

线程读Volatile变量的过程：

1. 从主内存中读取Volatile变量的最新值到线程的本地内存中；
2. 从本地内存中读取Volatile变量的副本

Volatile 实现内存可见性原理（了解）：

写操作时，通过在写操作指令后加入一条store屏障指令，让本地内存中变量的值能够刷新到主内存中；

读操作时，通过在读操作前加入一条load屏障指令，及时读取到变量在主内存的值

PS: 内存屏障（Memory Barrier）是一种CPU指令，用于控制特定条件下的重排序和内存可见性问题。Java编译器也会根据内存屏障的规则禁止重排序

volatile的底层实现是通过插入内存屏障，但是对于编译器来说，发现一个最优布置来最小化插入内存屏障的总数几乎是不可能的，所以，JMM采用了保守策略。如下：

1. StoreStore 屏障可以保证在volatile写之前，其前面的所有普通**写操作都已经刷新到主内存**中。
2. StoreLoad 屏障的作用是**避免volatile写与后面可能有的volatile读/写操作重排序**。
3. LoadLoad 屏障用来禁止处理器把上面的volatile读与下面的普通读重排序。
4. LoadStore 屏障用来禁止处理器把上面的volatile读与下面的普通写重排序。

java内存模型为了实现`volatile`可见性和**禁止指令重排**两个语义，使用如下内存屏障插入策略：

1. 每个`volatile`写操作前边插入`Store-Store`屏障，后边插入`Store-Load(全能)`屏障；
2. 每个`volatile`读操作前边插入`Load-Load`屏障和`Load-Stroe`屏障；

![](https://img2022.cnblogs.com/blog/2138872/202202/2138872-20220207202943624-1853087839.png)

 如图所示：写`volatile`屏障指令插入策略可以保证在`volatile`写之前，所有写操作都已经刷新到主存对所有处理器可见了。其后全能型屏障指令为了避免写`volatile`与其后`volatile`读写指令重排序。

![](https://img2022.cnblogs.com/blog/2138872/202202/2138872-20220207203108834-456330018.png)

 读`volatile`时，会在其后插入两条指令防止`volatile`读操作与其后的读写操作重排序。

## 二、原子性的问题

虽然Volatile 关键字可以让变量在多个线程之间可见，但是Volatile**不具备**原子性。

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

以上出现原子性问题的原因是count++并不是原子性操作。

count = 5 开始，流程分析：  
1. 线程1读取count的值为5  
2. 线程2读取count的值为5  
3. 线程2加1操作  
4. 线程2最新count的值为6  
5. 线程2写入值到主内存的最新值为6  
这个时候，线程1的count为5，线程2的count为6。如果切换到线程1执行，那么线程1得到的结果是6，写入到主内存的值还是6，现在的情况是对count进行了两次加1操作，但是主内存实际上只是加1一次

解决方案：

- 使用synchronized
- 使用ReentrantLock（可重入锁）
- 使用AtomicInteger（原子操作）

### 使用synchronized

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

### 使用ReentrantLock（可重入锁）

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

### 使用AtomicInteger（原子操作）

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

## 三、Volatile 适合使用场景

a）对变量的写入操作不依赖其当前值

不满足：number++、count=count*5等

满足：boolean变量、直接赋值的变量等

b）该变量没有包含在具有其他变量的不变式中

不满足：不变式 low\<up

总结：变量真正独立于其他变量和自己以前的值，在单独使用的时候，适合用volatile

## 四、synchronized和volatile比较

- volatile不需要加锁，比synchronized更轻便，不会阻塞线程
- synchronized既能保证可见性，又能保证原子性，而volatile只能保证可见性，无法保证原子性

与锁相比，Volatile 变量是一种非常简单但同时又非常脆弱的同步机制，它在某些情况下将提供优于锁的性能和伸缩性。如果严格遵循 volatile 的使用条件（变量真正独立于其他变量和自己以前的值 ） 在某些情况下可以使用 volatile 代替 synchronized 来优化代码提升效率。  

---
《阿里巴巴开发手册》中：

【参考】 volatile 解决多线程内存不可见问题。

对于一写多读，是可以解决变量同步问题，但是如果多写，同样无法解决线程安全问题。

如果是 count ++操作，使用如下类实现：AtomicInteger count = new AtomicInteger(); count . addAndGet( 1 );

如果是 JDK 8，推荐使用 LongAdder 对象，比 AtomicLong 性能更好 （ 减少乐观锁的重试次数 ） 。