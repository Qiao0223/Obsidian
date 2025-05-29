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
