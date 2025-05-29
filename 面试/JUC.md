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
