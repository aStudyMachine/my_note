# 十八、StampedLock：有没有比读写锁更快的锁？

## 1. StampedLock  概述

- Java 在 **1.8** 这个版本里，提供了一种叫 **StampedLock** 的锁，它的性能比读写锁要好 (基于乐观锁思想)。
-  **StampedLock** 与  **ReadWriteLock** 的区别 . 
  - 前者是后者的**子集**.
  - StampedLock 支持三种锁模式 :
    - 写锁
    - 悲观读锁
    - **乐观读**  `tryOptimisticRead()` (相比于普通的 ReadWriteLock , 最大区别主要在此. 该**操作是无锁的**, 仅是为了获取当前锁的版本号 . ) 
- **StampedLock** 的 **写锁** 和 **悲观读锁** 跟 普通的读写锁的读锁一样都是互斥的 (读写互斥) . 



## 2. 关键API 介绍

> StampedLock  的 **悲观读锁** 和 写锁 加锁都会返回一个 long 类型的 stamp . 解锁时需要传入加锁时返回的stamp , 否则会报错 .   

- `tryOptimisticRead()` : 乐观读 , 返回一个stamp , 相当于版本号. 无锁操作. 
- `validate(stamp)` : 校验stamp , 是否已被修改 . 
- `readLock()` : 悲观读锁 , 有锁操作 , 返回stamp.
- `unlockRead(stamp)` : 悲观读锁解锁 . 
- `writeLock()` : 写锁加锁 , 返回stamp  .  
- `unlockWrite(stamp)` : 写锁解锁 .

- 悲观读示例 : 

  ```java
  class Point {
    private int x, y;
    final StampedLock sl = new StampedLock();
    
     // 计算到原点的距离  
    int distanceFromOrigin() {
      // 乐观读
      long stamp = sl.tryOptimisticRead();
      // 读入局部变量，
      // 读的过程数据可能被修改
      int curX = x, curY = y;
      // 判断执行读操作期间，
      // 是否存在写操作，如果存在，
      // 则 sl.validate 返回 false
      if (!sl.validate(stamp)){
        // 升级为悲观读锁
        stamp = sl.readLock();
        try {
          // 注意: 前面加了悲观读锁, 这里拿到的 x y 是被修改过之后的数据 (happens-before)
          curX = x;
          curY = y;
        } finally {
          // 释放悲观读锁
          sl.unlockRead(stamp);
        }
      }
      return Math.sqrt(
        curX * curX + curY * curY);
    }
  }
  ```

  



## 3. 注意点

- StampedLock **不支持重入**。
- 如果线程阻塞在 `StampedLock `的` readLock()` 或者 `writeLock()` 上时，此时调用该阻塞线程的` interrupt()` 方法，会导致 CPU 飙升。其他线程再次获取悲观读锁或者写锁时 , 也会被阻塞. 



## 4. 使用模板

- 读模板 : 

  ```java
  final StampedLock sl = new StampedLock();
   
  // 乐观读
  long stamp = sl.tryOptimisticRead();
  
  // 读方法局部变量
  // 省略业务逻辑......
  
  // 校验 stamp
  if (!sl.validate(stamp)){
    // stamp 校验不通过, 其他线程已执行写操作. 
    // 升级为悲观读锁 
    stamp = sl.readLock();
    try {
      // 读入方法局部变量
      // .....
    } finally {
      // 释放悲观读锁
      sl.unlockRead(stamp);
    }
  }
  // 使用方法局部变量执行业务操作
  // 注意!!! 这里校验stamp通过后 , 此时有可能在开始使用局部变量执行业务之前, 成员数据已经被其他线程修改了,  也不是真正的互斥 , 如果业务保证准确性, 请使用其他互斥锁.  
  ......
  ```

- 写模板 : 

  ```java
  long stamp = sl.writeLock();
  try {
    // 写共享变量
    ......
  } finally {
    sl.unlockWrite(stamp);
  }
  ```

  

