# ART 线程优先级与锁公平性解析

本文分析了 Android Runtime (ART) 中 `synchronized` 锁的公平性，以及为什么解锁时不根据线程优先级进行调度。

## 1. 核心结论
在 ART 的实现中，`synchronized` 的解锁过程 **并不根据 Java 线程优先级** 来决定下一个获取锁的线程。ART 的锁机制在本质上是**非公平的 (Non-fair)**。

## 2. 不同锁状态下的行为

### 2.1 Thin Lock (轻量级锁)
*   **机制**：基于原子指令 CAS (Compare-And-Swap)。
*   **优先级影响**：零。当一个线程释放 Thin Lock（将 LockWord 设为 0）时，所有正在自旋或尝试进入的线程都会发起 CAS 操作。谁的指令先被 CPU 执行成功，谁就获得锁。这完全取决于硬件时序和 CPU 调度，不参考线程优先级。

### 2.2 Fat Lock (重量级锁)
*   **机制**：基于操作系统的 **Futex (Fast Userspace Mutex)**。
*   **唤醒顺序**：当 `Monitor::Unlock` 释放底层互斥量时，它会触发内核的唤醒动作。
*   **内核行为**：Linux 内核中的 Futex 等待队列通常遵循 **FIFO (先进先出)** 原则。虽然内核调度器在分配 CPU 时间片时会考虑优先级，但在“选择哪个线程脱离阻塞状态”这一环节，Java 优先级并不参与 Futex 队列的排序。

## 3. 为什么不引入优先级调度？

ART 团队未在 `synchronized` 中引入优先级唤醒主要基于以下考量：

1.  **性能开销**：
    要实现优先级唤醒，`Monitor` 必须在用户态维护一个优先级队列（如红黑树或堆）。这意味着每次入队（Wait/Contend）和出队（Notify/Unlock）都需要 $O(\log N)$ 的复杂度，而当前的 FIFO/CAS 逻辑接近 $O(1)$。对于高频的同步操作，这种开销是不可接受的。
2.  **优先级反转 (Priority Inversion)**：
    如果高优先级线程 A 等待低优先级线程 B 持有的锁，而中优先级线程 C 持续占用 CPU，会导致 A 永远无法运行。要解决这个问题需要实现 **优先级继承 (Priority Inheritance)** 协议，这会极大地增加运行时系统的复杂度和不稳定性。
3.  **内核与用户态的一致性**：
    ART 依赖宿主操作系统（Linux）进行线程调度。如果在用户态强行干预唤醒顺序，可能会与内核的负载均衡和 CFS 调度算法产生冲突，导致整体系统吞吐量下降。

## 4. 相关代码位置

*   `runtime/monitor.cc`: 包含 `Monitor::Unlock` 和 `SignalWaiterAndReleaseMonitorLock` 的逻辑。
*   `runtime/base/mutex.cc`: ART 对底层同步原语的封装，可以看到它如何使用 Futex。

## 5. 开发者建议

如果业务场景确实需要根据优先级或特定顺序来管理锁，建议：

1.  **使用 `ReentrantLock`**：
    ```java
    // 构造函数传入 true 开启公平锁
    Lock lock = new ReentrantLock(true);
    ```
    虽然公平锁会降低吞吐量，但它能保证等待时间最长的线程优先获得锁。
2.  **自定义优先级队列**：
    使用 `PriorityBlockingQueue` 等并发容器在应用层管理任务顺序，而不是依赖底层的同步锁。
