# 一、可见性、原子性和有序性问题：并发编程Bug的源头

## 1.  并发编程的幕后故事

### 1.1. 源头之一 #:   CPU、内存、I/O 设备 三者的速度差异

- 三者之间存在速度差异 **=>** 故引入**缓存机制** . **=>** 缓存导致的**可见性问题**

- 一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称为**可见性**。

- 多核(多cpu)时代  , 每个cpu 都有自己独立的缓存 ,  cpu对数据的读写总是将数据放入自己的缓存中操作 , 不同的cpu操作同一个数据对象时 , 各自的缓存对于对方来说 , 都是不可见的 , 这就是可见性问题 .  如图所示 : 

  ![多核 CPU 的缓存与内存关系图](http://assets.studymachine.cn/img/202110311643483.png)

- 代码示例 : 

  ```java
  public class Test {
  
      private static long count = 0;
  
      private void add10K() {
          int idx = 0;
          while (idx++ < 10000) {
              count += 1;
          }
      }
  
      /**
       * 开启两个线程, 各自分别对公共变量 count 执行 10000次 ++1 操作
       * 直觉上最后打印的结果应该是  20000 , 实际上由于各个线程各自维护的缓存问题 , 最终打印的结果 会 <= 20000
       */
      public static long calc() throws InterruptedException {
          final Test test = new Test();
  
          // 创建两个线程，执行 add() 操作
          Thread th1 = new Thread(test::add10K);
  
          Thread th2 = new Thread(test::add10K);
  
          // 启动两个线程
          th1.start();
          th2.start();
  
          // 等待两个线程执行结束
          th1.join();
          th2.join();
  
          return count;
      }
  
      public static void main(String[] args) throws InterruptedException {
          long calc = calc();
          System.out.println("calc = " + calc);
      }
  }
  
  ---------------------  打印结果 -----------------------
  calc = 17132
  ```





### 1.2. 源头之二：线程切换带来的原子性问题

- 背景 : 我们现在基本都使用高级语言编程，高级语言里一条语句往往需要多条 CPU 指令完成，例如上面代码中的`count += 1`，至少需要三条 CPU 指令。
  - 指令 1：首先，需要把变量 count 从内存加载到 CPU 的寄存器；
  - 指令 2：之后，在寄存器中执行 +1 操作；
  - 指令 3：最后，将结果写入内存（缓存机制导致可能写入的是 CPU 缓存而不是内存）。

- 所以 , 就CPU的层面上 , 高级语言执行的一条语句 , 并不是原子性(**我们把一个或者多个操作在 CPU 执行的过程中不被中断的特性称为原子性**)的. 
- 非原子操作的执行路径示意图 : 
 ![非原子操作的执行路径示意图](http://assets.studymachine.cn/img/202111020040520.png)

 - 两个线程同时对 count + 1操作 , 由于A +1未完成 , 线程切换到B , B +1 完成 写入内存 , 此时count = 1 , 线程切换回A , A中缓存的结果是 count = 0 ,  此时执行 + 1 操作 , count = 0 + 1 = 1 , 写入内存 , 导致最终的结果是1 , 而不是2 . 





### 1.3. 源头之三：编译优化带来的有序性问题

- 编译器为了优化性能，有时候会**改变程序中语句的先后顺序**，例如程序中：“a=6；b=7；”编译器优化后可能变成“b=7；a=6；”

  - 经典示例 :  Java 双重检查创建单例对象 . 

    ```java
    public class Singleton {
      static Singleton instance;
      static Singleton getInstance(){
        if (instance == null) {
          synchronized(Singleton.class) {
            if (instance == null)
                // 问题关键点 new 操作
              instance = new Singleton();
            }
        }
        return instance;
      }
    }
    ```

    - 问题的关键点主要在 `new Singleton();` 上 .  new 的底层操作顺序被优化

    - 我们所理解的 new 一个对象的过程 , 编译器所进行的操作顺序是  : 

      ```
      1. 分配一块内存 M；
      2. 在内存 M 上初始化 Singleton 对象；
      3. 然后 M 的地址赋值给 instance 变量。
      ```

      实际上却是 : 

      ```
      1. 分配一块内存 M
      2. 然后 M 的地址赋值给 instance 变量
      3. 在内存 M 上初始化 Singleton 对象 
      ```

    

  - 被优化顺序后的整个执行顺序如下 : 

    ![双重检查创建单例的异常执行路径](http://assets.studymachine.cn/img/202111160001528.png)

    >  这样就会出现一个问题 , instance 被赋予一个未初始化实际对象的内存地址 ,  然后其他线程在判断 instance 是否为 null 时 , 就是判断通过  , 导致创建多一个对象 , 违背单例.