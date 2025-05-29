Java JUC（Java Util Concurrent）是 Java 5 引入的并发工具包，旨在简化多线程编程，提升程序在多核处理器上的性能和可靠性。它提供了丰富的类和接口，涵盖线程管理、同步控制、并发容器、原子操作等多个方面。

# 1. 核心模块

### 1. **线程池（Executors）**

线程池通过重用线程来提高资源利用率，避免频繁创建和销毁线程的开销。

- `ExecutorService`：提供了管理和控制线程池的方法，如 `submit()` 和 `shutdown()`。
- `ThreadPoolExecutor`：线程池的核心实现类，允许自定义线程池的参数。
- `ScheduledExecutorService`：支持任务的定时和周期性执行。

### 2. **锁机制（Locks）**

JUC 提供了比 `synchronized` 更灵活的锁机制。

- `ReentrantLock`：可重入锁，支持公平锁和非公平锁选择。
- `ReentrantReadWriteLock`：读写锁，允许多个读线程同时访问，但写线程独占。
- `StampedLock`：支持乐观读锁，提高读操作的性能。

### 3. **原子变量类（Atomic）**

原子类提供了对基本数据类型和对象的原子操作，避免使用锁。

- `AtomicInteger`、`AtomicLong`、`AtomicBoolean`：对基本类型的原子操作。
- `AtomicReference`：对对象引用的原子操作。
- `AtomicStampedReference`：解决 ABA 问题的原子引用。

### 4. **并发集合（Concurrent Collections）**

线程安全的集合类，适用于并发环境。

- `ConcurrentHashMap`：高效的线程安全哈希表。
- `CopyOnWriteArrayList`、`CopyOnWriteArraySet`：适用于读多写少的场景。
- `ConcurrentLinkedQueue`、`ConcurrentLinkedDeque`：基于链表的非阻塞队列和双端队列。

### 5. **同步辅助类（Synchronizers）**

用于线程间的协调和控制。

- `CountDownLatch`：允许一个或多个线程等待其他线程完成操作。
- `CyclicBarrier`：使一组线程互相等待，直到到达某个公共屏障点。
- `Semaphore`：控制同时访问特定资源的线程数量。
- `Exchanger`：用于两个线程之间交换数据。

### 6. **Fork/Join 框架**

用于并行执行任务，将大任务拆分为小任务，提高处理效率。

- `ForkJoinPool`：执行任务的线程池。
- `RecursiveTask`、`RecursiveAction`：表示可分解的任务。

# 2. AQS（AbstractQueuedSynchronizer，抽象队列同步器）

Java 中的 AQS（AbstractQueuedSynchronizer，抽象队列同步器）是构建高性能并发同步器的核心框架，广泛应用于 ReentrantLock、CountDownLatch、Semaphore 等类的实现中。它通过维护一个同步状态（state）和一个先进先出（FIFO）的等待队列，协调多个线程对共享资源的访问。

## 2.1. 组成

### 1. 同步状态（state）

AQS 使用一个 `volatile int state` 变量表示同步状态。通过 `getState()`、`setState(int)` 和 `compareAndSetState(int, int)` 方法对其进行原子操作，确保线程间的可见性和安全性。

### 2. 同步队列（Sync Queue）

当线程获取资源失败时，AQS 会将该线程封装成一个节点（Node），并加入到一个基于 CLH（Craig, Landin, and Hagersten）队列变体的双向链表中，形成同步队列。该队列维护了等待获取锁的线程顺序，确保线程按 FIFO 原则获取锁。

### 3. Node 节点结构

每个 Node 包含以下关键字段：
- `Thread thread`：表示当前节点对应的线程。
- `Node prev` 和 `Node next`：指向前驱和后继节点，形成双向链表。
- `int waitStatus`：表示节点的等待状态，如：
    - `0`：默认状态。
    - `SIGNAL (-1)`：表示后继节点需要被唤醒。
    - `CANCELLED (1)`：表示线程取消了等待。
    - `CONDITION (-2)`：表示节点在条件队列中等待。
    - `PROPAGATE (-3)`：用于共享模式下的传播机制。

## 2.2. 独占与共享模式

### 1. 独占模式（Exclusive）

在独占模式下，只有一个线程能获取到同步状态，其他线程需等待。例如，ReentrantLock 就是基于独占模式实现的。

### 2. 共享模式（Shared）

在共享模式下，多个线程可以同时获取同步状态，适用于读操作等场景。例如，Semaphore 和 CountDownLatch 使用共享模式。

## 2.3. 条件队列（Condition Queue）

AQS 提供了内部类 `ConditionObject`，用于实现条件变量机制。当线程调用 `await()` 方法时，会释放当前锁，并进入条件队列等待；当其他线程调用 `signal()` 或 `signalAll()` 方法时，会将条件队列中的线程转移回同步队列，重新竞争锁。

## 2.4. 工作流程概述

- **获取锁**：线程尝试通过 `tryAcquire()` 或 `tryAcquireShared()` 方法获取锁。
- **入队等待**：获取失败的线程被封装成 Node，加入同步队列，并阻塞等待。
- **释放锁**：持有锁的线程调用 `release()` 或 `releaseShared()` 方法释放锁，唤醒队列中的下一个线程。
- **条件等待**：线程在条件变量上调用 `await()` 方法，进入条件队列等待特定条件。
- **条件唤醒**：其他线程调用 `signal()` 或 `signalAll()` 方法，唤醒条件队列中的线程，使其重新进入同步队列竞争锁。

# 3. CAS（Compare-And-Swap，比较并交换）

CAS（Compare-And-Swap，比较并交换）是 Java 并发编程中实现原子操作的核心机制之一，广泛应用于 `java.util.concurrent.atomic` 包中的原子类（如 `AtomicInteger`、`AtomicReference`）以及 AQS（AbstractQueuedSynchronizer）等同步器的底层实现。

## 3.1. 原理

CAS 是一种无锁的原子操作机制，其基本原理是：

1. 读取某个变量的当前值（V）。
2. 将该值与预期值（E）进行比较。
3. 如果当前值等于预期值，则将该变量的值更新为新值（N）；否则，不进行任何操作。

整个过程是原子的，即在执行期间不会被其他线程中断。

## 3.2. Java 中的 CAS 实现

在 Java 中，CAS 操作主要通过 `sun.misc.Unsafe` 类提供的方法实现，例如
```
public final native boolean compareAndSwapInt(Object obj, long offset, int expect, int update);
```
这些方法底层调用了 CPU 提供的原子指令（如 x86 架构中的 `cmpxchg` 指令）来实现原子性操作，确保在多线程环境下的数据一致性。
例如，`AtomicInteger` 类的 `getAndIncrement()` 方法的实现如下
```
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```
其中，`getAndAddInt` 方法内部使用了 CAS 操作来确保原子性。

## 3.3. 优点

- **无锁机制**：避免了传统锁机制带来的线程阻塞和上下文切换，提高了程序的并发性能。
- **原子性保证**：通过硬件级别的原子指令，确保操作的原子性，避免了数据竞争问题。

## 3.4. 缺点

- **ABA 问题**：如果一个变量的值从 A 变为 B，再变回 A，CAS 操作会误认为该变量未被修改，可能导致逻辑错误。
    **解决方案**：使用 `AtomicStampedReference` 或 `AtomicMarkableReference` 等类，通过引入版本号或标记位来检测变量的变化。
    
- **自旋开销大**：在高并发环境下，CAS 操作可能会频繁失败，导致线程不断自旋重试，增加 CPU 开销。
    **优化策略**：引入自适应自旋或退避策略，减少无效的自旋操作。
    
- **只能操作单个变量**：CAS 操作只能保证对单个变量的原子性，无法直接处理多个变量的原子操作。
    **解决方案**：通过组合多个 CAS 操作或使用更高级的同步机制（如锁）来实现多个变量的原子性。

## 3.5. 应用场景

CAS 广泛应用于以下场景：

- **原子类的实现**：如 `AtomicInteger`、`AtomicLong` 等，提供了无锁的原子操作方法。
- **并发数据结构**：如 `ConcurrentLinkedQueue`、`ConcurrentHashMap` 等，利用 CAS 实现高效的并发操作。
- **AQS 同步器**：如 `ReentrantLock`、`CountDownLatch` 等，底层通过 CAS 实现线程的同步和协调。

# 4. 死锁

死锁是指两个或多个线程在执行过程中，因争夺资源而造成的一种互相等待的现象。具体来说，线程 A 持有资源 1，等待资源 2；而线程 B 持有资源 2，等待资源 1。由于相互等待，导致所有线程都无法继续执行。

## 4.1. 死锁的四个必要条件

1. **互斥条件**：一个资源每次只能被一个线程使用。
2. **请求与保持条件**：一个线程因请求资源而阻塞时，对已获得的资源保持不放。
3. **不可剥夺条件**：线程已获得的资源，在未使用完之前，不能被其他线程强行剥夺。
4. **循环等待条件**：若干线程之间形成一种头尾相接的循环等待资源关系。

只有同时满足以上四个条件，才可能发生死锁。

示例代码：死锁的产生
```
public class DeadlockExample {
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        Thread thread1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("Thread 1: Holding lockA...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (lockB) {
                    System.out.println("Thread 1: Holding lockB...");
                }
            }
        });

        Thread thread2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("Thread 2: Holding lockB...");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                synchronized (lockA) {
                    System.out.println("Thread 2: Holding lockA...");
                }
            }
        });

        thread1.start();
        thread2.start();
    }
}
```
在上述代码中，两个线程分别持有 `lockA` 和 `lockB`，并试图获取对方的锁，导致死锁。

## 4.2. 避免死锁

1. **避免嵌套锁**：尽量避免一个线程同时持有多个锁。
2. **使用 `tryLock` 方法**：`ReentrantLock` 提供了 `tryLock()` 方法，可以设置超时时间，避免无限期等待。
```
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // 执行任务
    } finally {
        lock.unlock();
    }
} else {
    // 获取锁失败，执行其他操作
}
```
3. **统一锁的获取顺序**：确保所有线程以相同的顺序获取锁，避免循环等待。  
4. **使用 `ReentrantLock` 的可中断特性**：通过 `lockInterruptibly()` 方法，可以在等待锁的过程中响应中断，避免死锁。  

## 4.3. 检测死锁

### 1. 使用 `jstack` 命令

`jstack` 是 JDK 提供的命令行工具，可用于生成 Java 线程的堆栈信息。通过以下步骤可以检测死锁：

1. 使用 `jps` 命令获取目标 Java 进程的 PID。
2. 执行 `jstack -l <PID>` 命令，查看线程堆栈信息。
3. 在输出中查找 `Found one Java-level deadlock:` 关键词，若存在，说明发生了死锁。

例如：
```
jps
jstack -l 12345
```

### 2. 使用 JConsole 工具

JConsole 是 JDK 提供的图形化监控工具，可用于监控 Java 应用程序的性能和资源使用情况。检测死锁的步骤如下：[掘金+1知乎专栏+1](https://juejin.cn/post/7019476990302355470?utm_source=chatgpt.com)

1. 启动 JConsole，连接到目标 Java 应用程序。
2. 切换到“线程”选项卡。
3. 点击“检测死锁”按钮，若存在死锁，JConsole 会显示相关线程信息。

### 3. 使用 VisualVM 工具

VisualVM 是一个功能强大的可视化工具，可用于分析和监控 Java 应用程序。检测死锁的步骤如下：

1. 启动 VisualVM，连接到目标 Java 应用程序。
2. 切换到“线程”选项卡。
3. 点击“线程 Dump”按钮，查看线程堆栈信息，若存在死锁，VisualVM 会显示相关提示。

# 5. Volatile

在 Java 并发编程中，`volatile` 关键字是一个轻量级的同步机制，主要用于确保变量在多线程环境下的**可见性**和**有序性**，但**不保证原子性**。

## 5.1. 核心特性

### 1. 可见性（Visibility）

当一个线程修改了被 `volatile` 修饰的变量，新的值会立即被刷新到主内存中，其他线程在读取该变量时会直接从主内存中获取最新的值，而不是从各自的工作内存中读取旧值。

例如，以下代码中，如果不使用 `volatile`，线程 B 可能无法感知线程 A 对 `flag` 的修改，从而导致无限循环：[博客园+1GitHub+1](https://www.cnblogs.com/flydean/p/12680283.html?utm_source=chatgpt.com)
```
public class VisibilityExample {
    private static volatile boolean flag = false;

    public static void main(String[] args) {
        new Thread(() -> {
            while (!flag) {
                // 等待 flag 变为 true
            }
            System.out.println("Flag is true.");
        }).start();

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        flag = true;
    }
}
```
在上述示例中，`volatile` 保证了线程对 `flag` 变量的可见性，避免了线程间的可见性问题。

### 2. 有序性（Ordering）

`volatile` 关键字会禁止指令重排序优化，确保代码执行的顺序符合程序的预期。具体来说，写入 `volatile` 变量的操作会在其之前的所有操作执行完成后进行，读取 `volatile` 变量的操作会在其之后的所有操作开始之前进行。

这在实现如双重检查锁定（Double-Checked Locking, DCL）等模式时非常重要。
```
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
在上述 DCL 实现中，`volatile` 确保了 `instance` 的初始化过程不会被指令重排序，从而避免了返回未完全初始化对象的问题。

### 3. 不保证原子性（Atomicity）

虽然 `volatile` 保证了变量的可见性和有序性，但它**不保证操作的原子性**。例如，`i++` 操作实际上包含读取、修改和写入三个步骤，即使 `i` 被声明为 `volatile`，多个线程同时执行 `i++` 仍可能导致竞态条件。

要实现原子性操作，可以使用 `synchronized` 关键字或原子类（如 `AtomicInteger`）。
```
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // 原子性自增
```

## 5.2. 使用场景

- **状态标志**：如用于控制线程停止的标志变量。
- **单例模式**：配合 DCL 实现线程安全的懒加载单例。
- **事件通知**：在一个线程中修改变量，其他线程能够立即感知变化。

## 5.3. 注意事项

- **不可用于复合操作**：如 `i++`、`i = i + 1` 等操作不是原子性的，`volatile` 无法保证其线程安全。
- **不适用于依赖变量当前值的操作**：如检查变量是否为特定值后再修改的操作，可能会出现竞态条件。
- **不替代锁机制**：在需要保证原子性或更复杂的同步控制时，应使用 `synchronized` 或其他并发控制机制。

## 5.4. 底层实现原理

### 1. 保证可见性：MESI 协议与缓存一致性

在多核处理器系统中，每个核心都有自己的缓存。当一个线程修改了被 `volatile` 修饰的变量时，JMM 会确保该修改立即写入主内存，并使其他线程的缓存中该变量的副本失效。

这一机制依赖于 CPU 的缓存一致性协议，常见的如 MESI（Modified、Exclusive、Shared、Invalid）协议。当一个核心修改了某个变量，它会通过总线嗅探（Bus Snooping）机制通知其他核心将该变量的缓存标记为无效，从而确保其他线程读取到的是最新的值。

### 2. 保证有序性：内存屏障（Memory Barrier）

为了防止指令重排序带来的问题，JMM 在 `volatile` 变量的读写操作中插入内存屏障，强制执行特定的内存访问顺序：

- **写操作（Store）**：
    
    - 在写入 `volatile` 变量之前，插入 **StoreStore 屏障**，确保之前的写操作完成。
    - 在写入之后，插入 **StoreLoad 屏障**，防止后续的读写操作被重排序到前面。[博客园+1程序猿说你好+1](https://www.cnblogs.com/vipstone/p/18044839?utm_source=chatgpt.com)
    
- **读操作（Load）**：
    
    - 在读取 `volatile` 变量之前，插入 **LoadLoad 屏障**，确保之前的读操作完成。
    - 在读取之后，插入 **LoadStore 屏障**，防止后续的写操作被重排序到前面。[程序猿说你好](https://monkeysayhi.github.io/2016/11/29/volatile%E5%85%B3%E9%94%AE%E5%AD%97%E7%9A%84%E4%BD%9C%E7%94%A8%E3%80%81%E5%8E%9F%E7%90%86/?utm_source=chatgpt.com)
    

这些内存屏障通过底层的 CPU 指令（如 x86 架构中的 `LOCK` 前缀）实现，确保了操作的有序性。

### 3. `volatile` 与 `happens-before` 关系

根据 JMM 的定义，对一个 `volatile` 变量的写操作 **happens-before** 于后续对该变量的读操作。这意味着：

- 一个线程对 `volatile` 变量的写操作，对其他线程的后续读操作是可见的。
- 确保了多线程环境下的内存可见性和操作有序性。

# 6. Synchronized

在 Java 并发编程中，`synchronized` 是一种内置的同步机制，用于控制多个线程对共享资源的访问，防止数据不一致和线程安全问题。

## 6.1. `synchronized` 的使用方式

`synchronized` 关键字可以用于以下三种场景：

1. **修饰实例方法**：作用于当前实例对象，进入同步方法前需要获得该实例的锁。
    
    `public synchronized void instanceMethod() {     // 同步代码 }`
    

2. **修饰静态方法**：作用于当前类对象，进入同步方法前需要获得该类的类锁。
    
    `public static synchronized void staticMethod() {     // 同步代码 }`
    

3. **修饰代码块**：指定加锁对象，对给定对象加锁。
    
    `public void method() {     synchronized (lockObject) {         // 同步代码     } }`
    

通过上述方式，`synchronized` 可以确保在同一时刻，只有一个线程可以执行被同步的代码块，从而实现线程间的互斥访问。

## 6.2. 特性

- **可重入性**：同一线程可以多次获得同一把锁，避免死锁。
- **原子性**：确保同步代码块中的操作是原子的，防止线程间的干扰。
- **可见性**：线程在释放锁之前，会将对共享变量的修改刷新到主内存，其他线程在获得锁之后，可以看到最新的共享变量值。
- **有序性**：在进入同步块之前，JVM 会插入内存屏障，禁止指令重排序，确保代码执行的顺序性。

## 6.3. 注意事项

- **避免死锁**：在设计同步代码时，注意锁的获取顺序，避免多个线程相互等待，导致死锁。
- **锁的粒度**：尽量缩小同步代码块的范围，减少锁的持有时间，提高并发性能。
- **性能考虑**：在高并发场景下，频繁的锁竞争可能导致性能下降，可考虑使用 `java.util.concurrent` 包中的锁机制，如 `ReentrantLock`。

## 6.4. 底层实现原理

在 JVM 中，`synchronized` 是通过**监视器锁（Monitor）**实现的，每个对象在内存中都有一个与之关联的 Monitor。

### 对象头与 Monitor

Java 对象在内存中的布局包括对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。对象头中包含了 Mark Word，用于存储对象的哈希码、GC 分代年龄、锁信息等。
当线程进入同步代码块时，会尝试获取对象的 Monitor。如果 Monitor 已被其他线程持有，当前线程将被阻塞，直到获得 Monitor。

### 锁的状态与升级

为了提高性能，JVM 对锁进行了优化，引入了多种锁状态，并支持锁的升级和降级：

1. **无锁（No Lock）**：默认状态，适用于不存在竞争的场景。
2. **偏向锁（Biased Locking）**：当一个线程访问同步块并获得锁时，会在对象头中记录线程 ID，之后该线程再次访问同步块时，无需再次加锁。
3. **轻量级锁（Lightweight Locking）**：当偏向锁被其他线程竞争时，会升级为轻量级锁，采用自旋的方式尝试获取锁，避免线程阻塞。
4. **重量级锁（Heavyweight Locking）**：当自旋失败或竞争激烈时，升级为重量级锁，线程会被阻塞，等待唤醒。[

锁的升级过程是单向的，即从无锁 → 偏向锁 → 轻量级锁 → 重量级锁，无法降级。

# 7. ReentrantLock

在 Java 并发编程中，`ReentrantLock` 是 `java.util.concurrent.locks` 包中提供的一个可重入的互斥锁，功能上类似于 `synchronized`，但提供了更高的灵活性和扩展性。

## 7.1. 特性

- **可重入性**：同一线程可以多次获取同一把锁，而不会发生死锁。
- **公平性**：默认情况下，`ReentrantLock` 是非公平锁，即线程获取锁的顺序不保证先来先得。可以通过构造函数设置为公平锁，确保线程按照请求锁的顺序获得锁。
- **可中断性**：线程在等待锁的过程中，可以响应中断，避免出现死锁的情况。
- **尝试获取锁**：提供 `tryLock()` 方法，线程可以尝试获取锁，若获取不到则立即返回，避免长时间阻塞。
- **超时获取锁**：提供 `tryLock(long timeout, TimeUnit unit)` 方法，线程在指定时间内尝试获取锁，超时未获取到则返回 `false`。
- **条件变量**：通过 `newCondition()` 方法获取 `Condition` 对象，实现线程间的通信，比 `synchronized` 的 `wait()` 和 `notify()` 更加灵活。

## 7.2. 使用示例

```
import java.util.concurrent.locks.ReentrantLock;

public class Counter {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock(); // 加锁
        try {
            count++;
        } finally {
            lock.unlock(); // 解锁
        }
    }

    public int getCount() {
        return count;
    }
}
```

## 7.3. 高级功能

### 1. 公平锁与非公平锁

创建 `ReentrantLock` 实例时，可以指定锁的公平性：
```
ReentrantLock fairLock = new ReentrantLock(true); // 公平锁
ReentrantLock nonFairLock = new ReentrantLock();  // 非公平锁（默认）
```
公平锁保证线程按照请求锁的顺序获得锁，避免线程饥饿；非公平锁可能导致某些线程长时间得不到锁，但性能通常较好。

### 2. 可中断锁

使用 `lockInterruptibly()` 方法，线程在等待锁的过程中可以响应中断
```
try {
    lock.lockInterruptibly();
    // 执行任务
} catch (InterruptedException e) {
    // 处理中断
} finally {
    lock.unlock();
}
```

### 3. 尝试获取锁

使用 `tryLock()` 方法，线程尝试获取锁，若获取不到则立即返回
```
if (lock.tryLock()) {
    try {
        // 执行任务
    } finally {
        lock.unlock();
    }
} else {
    // 未获取到锁，执行其他操作
}
```
使用 `tryLock(long timeout, TimeUnit unit)` 方法，线程在指定时间内尝试获取锁
```
try {
    if (lock.tryLock(1, TimeUnit.SECONDS)) {
        try {
            // 执行任务
        } finally {
            lock.unlock();
        }
    } else {
        // 超时未获取到锁，执行其他操作
    }
} catch (InterruptedException e) {
    // 处理中断
}
```

### 4. 条件变量

`ReentrantLock` 提供 `newCondition()` 方法获取 `Condition` 对象，实现线程间的通信
```
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedBuffer {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition(); 
    private final Condition notEmpty = lock.newCondition(); 

    private final Object[] items = new Object[100];
    private int putptr, takeptr, count;

    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```
在上述示例中，`Condition` 对象 `notFull` 和 `notEmpty` 分别用于控制缓冲区的满和空状态，实现了生产者-消费者模型。

## 7.4. 与 `synchronized` 的比较
|特性|`synchronized`|`ReentrantLock`|
|---|---|---|
|可重入性|是|是|
|公平性|否|可选（默认否）|
|可中断性|否|是|
|尝试获取锁|否|是|
|超时获取锁|否|是|
|条件变量|否|是|
|锁的释放|自动|手动|
|性能|较低|较高|

## 7.5. 实现机制

### 1. 基于 AQS 的同步控制

`ReentrantLock` 通过继承自 AQS 的内部类 `Sync` 来实现锁的获取与释放。AQS 维护了一个 `state` 变量表示锁的状态（0 表示未锁定，正数表示锁定次数），并使用一个 FIFO 队列（CLH 队列的变体）来管理等待获取锁的线程。

### 2. 锁的获取流程

- **非公平锁**（默认）：
    
    - 线程尝试通过 CAS（Compare-And-Swap）操作将 `state` 从 0 设置为 1，成功则获得锁。
    - 若失败，则进入 AQS 的同步队列，等待前驱节点释放锁后被唤醒。
        
- **公平锁**：
    
    - 线程在尝试获取锁前，会检查同步队列中是否有其他线程等待，若有，则排队等待；否则，尝试通过 CAS 获取锁。
        

### 3. 可重入性支持

`ReentrantLock` 支持可重入性，即同一线程可以多次获取同一把锁。每次获取锁时，`state` 值加 1；每次释放锁时，`state` 值减 1；当 `state` 为 0 时，锁被完全释放。

### 4. 锁的释放流程

释放锁时，`state` 值减 1。如果 `state` 为 0，表示锁已完全释放，此时会唤醒同步队列中的下一个等待线程。


