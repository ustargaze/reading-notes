# Java 并发编程之美

## 第一部分 Java 并发编程基础篇

### 第 1 章 并发编程线程基础

**什么是进程？什么是线程？**

> 进程是代码在数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，线程则是进程的一个执行路径，一个进程中至少有一个线程，进程中的多个线程共享进程的资源。
>
> 操作系统在分配资源时是把资源分配给进程的，但是CPU资源比较特殊，它是被分配到线程的，因为真正要占用CPU运行的是线程，所以也说线程是CPU分配的基本单位。

❓：这里把进程当作是资源分配和调度的基本单位，线程则是 CPU 分配的基本单位。个人对于这个表述存在些许疑问，因为调度应该是指 CPU 调度也就是 CPU 资源分配的过程，把进程当成调度的基本单位应该不合适。

🤔 翻了一下[操作系统](https://weread.qq.com/web/reader/81d32df05a5d7d81deef204?)这本书，调度不仅仅只有 CPU 调度，我之前的理解应该有误，比如针对使用者占用不同资源的调度可以分为：高级调度、中级调度、低级调度，并且作业、进程、线程在不同的时期有不同的含义，尽管如此还是觉得说进程是调度的基本单位这种表述有些模糊。

目前暂时没办法清楚的说出各种调度的区别，还需要加强操作系统相关概念的学习，这里先暂留个疑问。😐 目前先认为<u>进程是代码在数据集合上的一次运行活动，是系统进行资源分配的基本单位，线程则是进程的一个执行路径，是CPU分配的基本单位，一个进程中至少有一个线程，进程中的多个线程共享进程的资源。</u>



**创建线程的方式**

> Java中有三种线程创建方式，分别为实现Runnable接口的run方法，继承Thread类并重写run的方法，使用FutureTask方式。

个人还是比较倾向 JDK 源码注释中所说的

> There are two ways to create a new thread of execution. One is to declare a class to be a subclass of <code>Thread</code>. This subclass should override the <code>run</code> method of class <code>Thread</code>. An instance of the subclass can then be allocated and started. 
>
> The other way to create a thread is to declare a class that implements the <code>Runnable</code> interface. That class then implements the <code>run</code> method. An instance of the class can then be allocated, passed as an argument when creating <code>Thread</code>, and started. 

创建一个新的执行线程有两种方式：

1. 实现一个 Thread 的子类重写 run 方法；
2. 声明一个类实现 Runnable 接口并实现 run 方法，然后创建 Thread 实例时，将此类的实例作为参数传递。

我觉得这个重点应带不是线程，而是执行，不管是重写子类也还还是实现 Runnable 接口也好，都是实现一个线程任务。Thread 中 run 方法的默认实现是如果 target （Runnable 类型）不为空则调用 target.run()。

总结一下

1. 创建线程的执行任务：上面说的两种
2. 创建 Java 线程：new Thread 或者 new Thread 的子类

3. 创建内核线程 Thread#start0。参考[大家都说Java有三种创建线程的方式！并发编程中的惊天骗局！](https://mp.weixin.qq.com/s/NspUsyhEmKnJ-4OprRFp9g)



**线程通知与等待**

1. **wait 方法**

   三个重载的 wait 的方法签名如下

   ```java
   public final native void wait(long timeout) throws InterruptedException;
   public final native void wait(long timeout, int nanos) throws InterruptedException;
   public final native void wait() throws InterruptedException;
   ```

   以 wait(timeout) 为例进行说明，线程调用对象的 wait 方法后，将被阻塞挂起，直到：

   1. 其他线程调用了该对象的 notify() 或者 notifyAll() 方法；
   2. 其他线程调用了该线程的 interrupt() 方法，使该线程抛出 InterruptedException；
   3. 等待时间超过了 timeout 所指定的毫秒数，如果 timeout = 0 表示没有超时时间。

   其他两个重载函数的作用为：wait() 等同于 wait(0)，wait(long timeout, int nanos) 等同于 wait(nanos > 0 ? timeout + 1: timeout);

   

   **需要注意的是，如果调用 wait() 方法的线程没有事先获取该对象的监视器锁，则调用 wait() 方法时调用线程会抛出IllegalMonitorStateException异常。**

   **一个线程如何获取一个共享变量的监视器锁呢？**

   1. 执行synchronized同步代码块时，使用该共享变量作为参数。
   2. 调用该共享变量的方法，并且该方法使用了synchronized修饰。

   **什么是虚假唤醒？**

   当一个线程在等待某个条件满足时（例如，通过`wait()`方法等待），在某些情况下，该线程可能会在条件未满足的情况下被唤醒。这种情况被称为虚假唤醒。

   **如何避免虚假唤醒？**

   通过在 while 循环中调用 wait 方法进行处理。

   ```java
   synchronized (lock) {
       while (!condition) { // 检查条件是否满足
           try {
               lock.wait(); // 如果条件未满足，则等待
           } catch (InterruptedException e) {
               // 处理中断异常
           }
       }
       // 条件满足，执行相应操作
   }
   ```

2. **notify / notifyAll 方法**

   线程调用共享对象的 notify() 方法后，会唤醒一个在该共享变量上调用 wait 系列方法后被挂起的线程，一个共享变量上可能会有多个线程在等待，具体唤醒哪个等待的线程是随机的。而 notifyAll() 方法则会唤醒所有在该共享变量上由于调用 wait 系列方法而被挂起的线程。

   此外，被唤醒的线程不能马上从 wait 方法返回并继续执行，它必须在获取了共享对象的监视器锁后才可以返回，也就是唤醒它的线程释放了共享变量上的监视器锁后，被唤醒的线程也不一定会获取到共享对象的监视器锁，这是因为该线程还需要和其他线程一起竞争该锁，只有该线程竞争到了共享变量的监视器锁后才可以继续执行。



**Thread#join 方法：等待线程执行终止**

三个重载的 join 的方法签名如下

```java
public final synchronized void join(long millis) throws InterruptedException;
public final synchronized void join(long millis, int nanos) throws InterruptedException;
public final void join() throws InterruptedException;
```

join 方法通过 synchronized 修饰，然后内部调用循环调用 Thread 对象的 wait 方法实现直到 Thread 对象所代表的线程死了，线程终止后会调用 Thread 对象的 notifyAll() 唤醒所有因调用 wait 方法而阻塞的线程，而线程因为调用 interrupt 被中断后 join 方法会抛出 InterruptedException 而返回（源自 wait）。

join(millis, nanos) 等同于 join(nanos >= 500000 || (nanos != 0 && millis == 0) ? ++millis : millis) 。

join() 等同于 join(0)。

所以，join 方法其实就是通过对 Thread 对象加锁，然后调用 wait 阻塞当前线程直至。



**Thread.sleep 方法：让线程睡眠**

Thread类中有一个静态的sleep方法，当一个执行中的线程调用了Thread的sleep方法后，调用线程会暂时让出指定时间的执行权，也就是在这期间不参与CPU的调度，但是该线程所拥有的监视器资源，比如锁还是持有不让出的。指定的睡眠时间到了后该函数会正常返回，线程就处于就绪状态，然后参与CPU的调度，获取到CPU资源后就可以继续运行了。如果在睡眠期间其他线程调用了该线程的interrupt（）方法中断了该线程，则该线程会在调用sleep方法的地方抛出InterruptedException异常而返回。



**Thread.yield 方法：让出 CPU 执行权**

yield 方法是 Thread 类中的一个静态方法，当一个线程调用 yield 时，是在暗示自己可以交出 CPU 使用权，但是线程调度器可以无条件忽略这个暗示。线程调用 yield 方法后，当前线程让出 CPU 使用权，然后处于就绪状态，线程调度器会从线程就绪队列里面获取一个线程优先级最高的线程，当然也有可能会调度到刚刚让出 CPU 的线程来获取 CPU 执行权。



**线程中断**

> 换个说法可能比较合适，Java 线程对象的中断标志。

Java中的线程中断是一种线程间的协作模式，通过设置线程的中断标志并不能直接终止该线程的执行，而是被中断的线程根据中断状态自行处理。

- interrupt 方法：Thread 类的实例方法，作用是中断线程，但是并不是真的中断线程，而是设置标志位。如果线程被调用了 interrupt 方法，线程因为调用了 wait 系列函数、join 方法或者 sleep 方法而被阻塞挂起的地方会抛出 Interrupted Exception 异常而返回。
- isInterrupted 方法：Thread 类的实例方法，检测线程是否被中断。
- interrupted 方法：Thread 类的静态方法，检测当前执行线程是否被中断，与 isInterrupted 不同的是，如果发现当前执行线程被中断，会将标志位清楚，也就是说变成非中断了。



**线程上下文切换**

线程上下文切换时机：

1. 线程正常执行完毕。
2. 线程未执行完，但是时间片用完。
3. 被其他线程打断。
4. 因为 I/O 之类的阻塞挂起。
5. 并发抢占锁或者资源时，线程没有抢到。
6. 代码致使线程挂起。例如sleep、join、wait方法的调用。



**线程死锁**

死锁是指两个或两个以上的线程在执行过程中，因争夺资源而造成的互相等待的现象。

<u>死锁产生的四个必要条件：互斥、请求并保持、非抢占、循环等待。</u>

死锁避免：破坏死锁的四个必要条件之一。



**守护线程和用户线程**

可以通过 Thread#setDaemon 方法传入 true 来将线程设置为守护线程。

守护线程与用户线程的区别是，当最后一个非守护线程结束时，JVM会正常退出，而不管当前是否有守护线程，也就是说守护线程是否结束并不影响JVM的退出。言外之意，只要有一个用户线程还没结束，正常情况下JVM就不会退出。



### 第 2 章 并发编程的其他基础知识

**Java 中的线程安全问题**

共享资源：被多个线程所持有或者说多个线程都可以去访问的资源。

线程安全问题是指当多个线程同时读写一个共享资源并且没有任何同步措施时，导致出现脏数据或者其他不可预见的结果的问题。



**伪共享**

为了解决计算机系统中主内存与CPU之间运行速度差问题，会在CPU与主内存之间添加一级或者多级高速缓冲存储器（Cache）。在 Cache 内部是按行存储的，其中每一行成为一个 Cache 行。Cache 行是 Cache 与主内存进行数据交换的单位，而一个 Cache 行可以存储多个变量。因此当多个线程同时修改一个缓存行里面的多个变量时，由于同时只能有一个线程操作缓存行，相比将每个变量放到一个缓存行，性能会有所下降，这就是伪共享。

如何避免伪共享？

- 在 JDK 8 之前一般都是通过字节填充的方式来避免该问题，也就是创建一个变量时使用填充字段填充该变量所在的缓存行，这样就避免了将多个变量存放在同一个缓存行中，

- JDK 8提供了一个sun.misc.Contended注解，用来解决伪共享问题。

  在默认情况下，@Contended注解只用于Java核心类，比如rt包下的类。如果用户类路径下的类需要使用这个注解，则需要添加JVM参数：-XX:-RestrictContended。填充的宽度默认为128，要自定义宽度则可以设置-XX:ContendedPaddingWidth参数。



**锁的概述**

1. 乐观锁与悲观锁
2. 公平锁与非公平锁
3. 独占锁和共享锁
4. 可重入锁
5. 自旋锁



## 第二部分 Java 并发编程高级篇

### 第 3 章 Java 并发包中 ThreadLocalRandom 原理剖析

**Random 类及其局限性**

> Math.random 使用的也是 Random 类。

使用 Random 生成随机数的需要种子，种子可以通过构造函数指定，如果不指定则会自动生成一个。

Random 生成随机数主要是两个步骤：

1. 根据老的种子来生成新的种子。
2. 根据新的种子来计算新的随机数。

在多线程场景下可能发生的问题：

1. 构造的多个 Random 实例，在未指定种子的情况下，内部生成的默认种子相同。
2. 使用同一个 Random 类，可能导致多个线程产生的新的种子是一样的。

对于问题 1，在 JDK 中使用如下代码解决：

```java
public Random() {
    this(seedUniquifier() ^ System.nanoTime());
}

private static long seedUniquifier() {
    // L'Ecuyer, "Tables of Linear Congruential Generators of
    // Different Sizes and Good Lattice Structure", 1999
    for (;;) {
        long current = seedUniquifier.get();
        long next = current * 181783497276652981L;
        if (seedUniquifier.compareAndSet(current, next))
            return next;
    }
}
```

对于问题 2，使用如下代码解决：

```java
protected int next(int bits) {
    long oldseed, nextseed;
    AtomicLong seed = this.seed;
    do {
        oldseed = seed.get();
        nextseed = (oldseed * multiplier + addend) & mask;
    } while (!seed.compareAndSet(oldseed, nextseed));
    return (int)(nextseed >>> (48 - bits));
}
```

总结来说就是通过 CAS + 自旋操作来保证多线程下随机数的随机性。但是这样当多个线程竞争同一个原子变量的更新操作，由于 CAS 只有一个线程会成功，会出现大量的自旋重试，这会降低并发性能。

**ThreadLocalRandom**

ThreadLocalRandom 为了避免多个线程对同一个原子变量的竞争操作，使用的是每个线程维护一个自己的种子变量的方式。

ThreadLocalRandom 是 <u>Random 的子类并且是一个饿汉式单例类</u>，通过 current 可以获取到实例对象。在初次调用 current 时会判断种子是否初始化，如果没有则会进行初始化。相关代码如下：

```java
static final void localInit() {
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    Thread t = Thread.currentThread();
    UNSAFE.putLong(t, SEED, seed);
    UNSAFE.putInt(t, PROBE, probe);
}

public static ThreadLocalRandom current() {
    if (UNSAFE.getInt(Thread.currentThread(), PROBE) == 0)
        localInit();
    return instance;
}
```

probeGenerator 和 seeder 是两个原子变量，用来为每个线程生成 threadLocalRandomProbe 和 threadLocalRandomSeed 这两个属性的值。这样对于原子变量更新的竞争只会出现在线程第一次调用 current 获取实例时，后续生成随机数并不会发生。



