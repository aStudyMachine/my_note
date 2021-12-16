# 十六、Semaphore：如何快速实现一个限流器？

## 1. 什么是信号量模型 ?

- 计算机科学家迪杰斯特拉（Dijkstra）于 1965 年提出 , 用于解决并发编程的模型. 



## 2. 信号箱模型的组成

- 一个计数器
- 一个等待队列
- 三个方法 `init()`、`down()` 和 `up() `
  - 这三个方法都是**原子性**的.
  - `init()`：设置计数器的初始值。
  - `down()`：计数器的值减 1；**如果此时计数器的值小于 0，则当前线程将被阻塞**，否则当前线程可以继续执行。
  - `up()`：计数器的值加 1；**如果此时计数器的值小于或者等于 0，则唤醒等待队列中的一个线程**，并将其从等待队列中移除。

- 信号量模型图

  ![](http://assets.studymachine.cn/img/202112111551247.png)



## 3. 如何使用信号量

- 信号量模型里面，down()、up() 这两个操作历史上最早称为 **P 操作**和 **V 操作**，所以信号量模型也被称为 **PV 原语**。

- 在 Java SDK 并发包里，`down()` 和 `up()` 对应的则是 `acquire()` 和`release()`。

- 使用信号量实现互斥 : 

  ```java
  static int count;
  
  // 初始化信号量
  static final Semaphore s  = new Semaphore(1);
  
  // 用信号量保证互斥    
  static void addOne() {
    s.acquire();  // 信号量 -1 => 相当于尝试加锁
    try {
      count+=1;
    } finally {
      s.release(); // 信号量 +1 => 相当于释放锁
    }
  }
  ```

  - **分析 :**  线程T1 执行  ` s.acquire()` 信号量 - 1 ,  如果 此时信号量 == 0 则继续往下走 .  后续线程 T2 执行   ` s.acquire()`  信号量 -1 . 此时 信号量  == -1 < 0 则阻塞 . 从而形成互斥 .  待 T1执行 `s.release()`  信号量 +1 == 0  符合 <= 0 则唤醒 T2 进入临界区  **(注意 : 这里T2不会重复去进行信号量 - 1)**





## 4. 快速实现一个限流器

- 在 Java SDK 中 , **Semaphore**  相对于 **Lock** , 有个很大的不同之处 .  **Semaphore 可以允许多个线程访问一个临界区**。

- 各种池化技术的思想 , 就可以看成一个限流器的落地 .  可以允许多个线程进入临界区 ,  同时 利用 Semaphore的阻塞功能 , 进行限流 .  

- 使用 `Semaphore` 实现限流器代码. 

  ```java
  // 对象池
  class ObjPool<T, R> {
    
     // 存放对象
    final List<T> pool;
      
    // 用信号量实现限流器
    final Semaphore sem;
      
    // 构造函数
    ObjPool(int size, T t){
        //  注意这里的对象池容器, 不能使用线程不安全的ArrayList 或者其他. 
      pool = new Vector<T>(){};
      for(int i=0; i<size; i++){
        pool.add(t);
      }
        // 注意!!! 这里入参 size, 就是初始化信号量 , 初始化信号量的数值 就是可以同时允许多少个线程进入临界区的数值 . 
      sem = new Semaphore(size);
    }
      
    // 利用对象池的对象，调用 func
    R exec(Function<T,R> func) {
      T t = null;
      sem.acquire();
      try {
        t = pool.remove(0);
        return func.apply(t);
      } finally {
        pool.add(t);
        sem.release();
      }
    }
  }
  
  // ========= 以上为定义 =======
  
  // ========= 以下为调用 ======= 
  
  // 创建对象池
  ObjPool<Long, String> pool = 
    new ObjPool<Long, String>(10, 2);
  
  // 通过对象池获取 t，之后执行  
  pool.exec(t -> {
      System.out.println(t);
      return t.toString();
  });
  ```

  > 简单总结 
  >
  > - 就是 池化思想  +  信号量(阻塞)   => 实现限流器. 
  > - 仅允许池的容量数量的对象同时进入临界区 , 也就是执行目标代码 . 