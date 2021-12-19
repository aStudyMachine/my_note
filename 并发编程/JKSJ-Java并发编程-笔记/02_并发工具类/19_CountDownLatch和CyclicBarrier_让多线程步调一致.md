

# 十九、CountDownLatch和CyclicBarrier：让多线程步调一致



## 1. 业务场景

- **场景** : 业务对账流程

- **描述** :  

  整个业务流程包含如下4个步骤 , 按以往同步的写法 , 这里的4个步骤应该是按顺序一步接一步执行执行的.  

  但是如下图可知 
  
  ![对账系统流程图](http://assets.studymachine.cn/img/202112191515705.png)
  

## 2. 优化


- **优化思路** :  

  步骤1 与 步骤2 其实互不依赖的 , 可以进一步优化为异步同时执行 .  整个优化的方向图例如下所示 
  
  ![image-20211219152526813](http://assets.studymachine.cn/img/202112191525900.png)

- **简陋实现** : 

  ```java
  while(存在未对账订单){
    // 查询未对账订单
    Thread T1 = new Thread(()->{
      pos = getPOrders();
    });
    T1.start();
    
      // 查询派送单
    Thread T2 = new Thread(()->{
      dos = getDOrders();
    });
    T2.start();
   
      // 等待 T1、T2 结束
    T1.join();
    T2.join();
    
      // 执行对账操作
    diff = check(pos, dos);
   
      // 差异写入差异库
    save(diff);
  } 
  ```





## 3. 二次优化

- **描述** : 线上系统整个对账操作不止一次 , 每次进行对账都要创建两个新的线程 , 也是一笔不小的开销 . 
- 优化思路 :  使用**线程池**.



### 3.1. 引入新的问题

- **描述** : 使用线程池之后 , 如果还是通过调用 `join()` 方法等待线程执行完成**退出**  , 是不可行的, 因为线程执行完之后 , 是不会退出的 , 而是再次回到线程池中 . 





###  3.2. 用 CountDownLatch 实现线程等待

- **描述** :  `CountDownLatch ` 是 Java 提供的现成的 线程等待 + 线程复用 的工具类. 

- **CountDownLatch  使用** : 

  ```java
  // 创建 2 个线程的线程池
  Executor executor = 
    Executors.newFixedThreadPool(2);
  
  while(存在未对账订单){
    // 计数器初始化为 2    // <======= 关键步骤 : 创建计数器, 初始化 = 2
    CountDownLatch latch =  new CountDownLatch(2);
      
    // 查询未对账订单
    executor.execute(()-> {
      pos = getPOrders();
      latch.countDown();  // <======= 关键步骤 , 计数器 -1
    });
      
    // 查询派送单
    executor.execute(()-> {
      dos = getDOrders();
      latch.countDown(); // <======= 关键步骤 , 计数器 -1
    });
    
    // 等待两个查询操作结束
    latch.await();    // <======= 关键步骤 , 等待两个 -1 操作的线程执行完毕
    
    // 执行对账操作
    diff = check(pos, dos);
      
    // 差异写入差异库
    save(diff);
  }
  ```

  

## 4. 进一步优化

### 4.1 优化思路

- **描述** : 

  两个查询操作查询完毕后 , 不用等待后续的两个更新操作 `check()`、`save()` 执行完之后 , 才去执行下一轮的两次查询操作 .  可以改为当前两次查询操作完成之后 , 直接执行下一轮查询 . 这需要**保证不同时间点查出来的两种不同数据一一对应**即可 .   (实际上就是一个 **生产者-消费者**的模型)

- **执行示意图** : 

  ![image-20211219165629974](http://assets.studymachine.cn/img/202112191656054.png)

### 4.2. 引入队列, 实现生产者-消费者模型

- **优化思路** : 

  - 两个查询订单的操作作为生产者 , 因为生产-消费各自是异步的 , 这里需要创建两个队列分别存放生产者查询的两种不同的订单数据 . 

  - 为了保证两个队列中的数据的是一一对应的.  这里需要利用**线程同步** , 即**两个查询订单的操作 , 先查询完成的操作 , 需要等待另一个查询操作查询完成后 , 再一起进入各自的队列** , 以此确保执行各自的出队操作时 , 数据都是对应的.  

    ![image-20211219174032069](http://assets.studymachine.cn/img/202112191740136.png)



### 4.3. 用 CyclicBarrier 实现线程同步

- **描述** : Java 并发包 提供的一个线程同步工具 **CyclicBarrier** . 
- 特性 : 
  - 创建 CyclicBarrier 时 , 会初始化一个**计数器** 以及 **回调函数** . 
  - 调用`await()` 来将计数器减 1 , 同时**等待计数器变成 0**
  - 当计数器减至0时, 会执行回调函数 . 
  -  执行完回调函数 , 计数器值还原到初始化值.  (自动重置功能)

- 代码示例 :  

  ```java
  // 订单队列, 必须要确保读写相互阻塞
  Vector<P> pos;
  
  // 派送单队列, 必须要确保读写相互阻塞
  Vector<D> dos;
  
  // 执行回调的线程池 , 这里线程池大小故意设置成1, 确保生产者是串行执行 . 也可以使用其他方式确 check() 中两个remove(0) 出队操作是原子的 , 避免出队时数据错乱 . 
  Executor executor = Executors.newFixedThreadPool(1);
  
  // 开启 CyclicBarrier 循环执行, 初始化为2, 需要等待计数器 至0
  final CyclicBarrier barrier =
    new CyclicBarrier(2, ()->{
      executor.execute(()->check());
    });
  
  // 开始生产数据.
  checkAll();
  
  // 消费者: 对账+写入差异库
  void check(){
    P p = pos.remove(0);
    D d = dos.remove(0);
    // 执行对账操作
    diff = check(p, d);
    // 差异写入差异库
    save(diff);
  }
  
  // 生产者 : 开启两个线程循环查询, 生产出各自的订单数据 . 
  void checkAll(){
    // 循环查询商品订单库
    Thread T1 = new Thread(()->{
      while(存在未对账订单){
        // 查询订单库
        pos.add(getPOrder());
        // 计数器-1 , 进入等待状态
        barrier.await();
      }
    });
    T1.start();  
      
    // 循环查询运输订单库
    Thread T2 = new Thread(()->{
      while(存在未对账订单){
        // 查询运单库
        dos.add(getDOrder());
        // 计数器-1 , 进入等待状态
        barrier.await();
      }
    });
    T2.start();
  }
  ```

  > 注意 : 这里代码并不完善 , 具体业务还需要具体设计整个逻辑. 也有不少需要优化的细节 . 



## 5. 总结

- `CountDownLatch `主要用来解决**一个线程等待多个线程**的场景 , `CyclicBarrier` 是**一组线程之间互相等待**. 
- `CountDownLatch`的计数器是不能循环利用的，也就是说一旦计数器减到 0，再有线程调用 `await()`，该线程会直接通过。`CyclicBarrier`**的计数器是可以循环利用的**，而且具备自动重置的功能，一旦计数器减到 0 会自动重置到你设置的初始值, CyclicBarrier 还可以**设置回调函数**. 

