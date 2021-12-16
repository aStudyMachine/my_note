# 十八、StampedLock：有没有比读写锁更快的锁？

## 1. StampedLock  概述

- Java 在 **1.8** 这个版本里，提供了一种叫 **StampedLock** 的锁，它的性能就比读写锁还要好。

-  **StampedLock** 与  **ReadWriteLock** 的区别 . 
  - 前者是后者的**子集**.
  - StampedLock 支持三种锁模式 :
    - 写锁
    - 悲观读锁
    - **乐观读**  `tryOptimisticRead()` (相比于普通的 ReadWriteLock , 最大区别主要在此.) 

- **StampedLock** 的 **写锁** 和 **悲观读锁** 跟 普通的读写锁的读锁和写锁是一样的. 
- 
