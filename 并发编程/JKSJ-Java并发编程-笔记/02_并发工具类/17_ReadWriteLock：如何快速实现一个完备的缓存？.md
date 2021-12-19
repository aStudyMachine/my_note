# 十七、ReadWriteLock：如何快速实现一个完备的缓存？

## 1. 读写锁概述

- 针对**读多写少**这种并发场景，Java SDK 并发包提供了读写锁——**ReadWriteLock**，非常容易使用，并且性能很好。
- 读写锁的三个基本守则 : 
  - 允许多个线程同时读共享变量；
  - 只允许一个线程写共享变量；
  - 如果一个写线程正在执行写操作，此时禁止读线程读共享变量。


## 2. 实现缓存的按需加载

- 先判断缓存中是否存在对应的缓存对象 , 没有则进行查库 , 并写入缓存中 . 

- 代码实现 : 

  ```java
  class Cache<K,V> {
      // 缓存容器
    final Map<K, V> m =
      new HashMap<>();
      
      // 读写锁
    final ReadWriteLock rwl = 
      new ReentrantReadWriteLock();
      
      // 读锁
    final Lock r = rwl.readLock();
      
      // 写锁
    final Lock w = rwl.writeLock();
   
      // 获取数据
    V get(K key) {
      V v = null;
      // 读缓存
      r.lock();    // 读锁加锁     ①
      try {
        v = m.get(key); ②
      } finally{
        r.unlock();  // 读锁解锁   ③
      }
      // 缓存中存在，返回
      if(v != null) {   ④
        return v;
      }  
      // 缓存中不存在，查询数据库
      w.lock();   // 写锁加锁      ⑤
      try {
        // 再次验证
        // 其他线程可能已经查询过数据库
        v = m.get(key); ⑥
        if(v == null){  ⑦
          // 查询数据库
          v= 省略代码无数
          m.put(key, v); // 放入缓存
        }
      } finally{
        w.unlock(); // 写锁解锁
      }
      return v;  // 返回数据
    }
  } 
  ```





## 3. 读写锁的升级与降级



### 3.1. 锁的升级

- 读锁加锁过程中 , 在释放读锁之前 , 进行写锁加锁 , 这个操作称作**读写锁的升级** . (读锁 ==> 写锁) . 
  - 注意 :  `ReadWriteLock `并不支持这种升级。 上述操作最终导致相关线程都被阻塞，永远也没有机会被唤醒。



### 3.2. 锁的降级

- 同理 , 由锁升级的定义可知 , 读写锁的降级 , **就是在写锁过程中(未释放写锁) , 进行读锁加锁 .** 

-  `ReadWriteLock ` 是支持锁的降级的 . 

- 代码示例 : 

  ```java
  class CachedData {
      
    Object data;
      
    volatile boolean cacheValid;
      
    final ReadWriteLock rwl =
      new ReentrantReadWriteLock();
      
    // 读锁  
    final Lock r = rwl.readLock();
      
    // 写锁
    final Lock w = rwl.writeLock();
    
    void processCachedData() {
      // 获取读锁
      r.lock();
      if (!cacheValid) {
        // 释放读锁，因为不允许读锁的升级
        r.unlock();
        // 获取写锁
        w.lock();
        try {
          // 再次检查状态  
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // 释放写锁前，降级为读锁
          // 降级是可以的
          r.lock(); ①  // 写锁中, 进行读锁加锁.
        } finally {
          // 释放写锁
          w.unlock(); 
        }
      }
      // 此处仍然持有读锁
      try {use(data);} 
      finally {r.unlock();}
    }
  }
  ```