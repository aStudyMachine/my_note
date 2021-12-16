# 二、内存模型：看Java如何解决可见性和有序性问题



## 1. Java 内存模型的处理方法

- cpu 缓存 => 可见性问题
- 编译优化 => 有序性问题
- 最简单粗暴的解决办法 : 全盘**禁用cpu 缓存**  && **禁用编译优化** .  **(不可取 !! 会导致程序性能极低 )** 
- 合理的办法 :  **按需**进行上述禁用
- JVM 如何按需进行禁用缓存以及指令优化 ? 
  - **volatile**
  - **synchronized** 
  - **final** 
  - 六项 **Happens-Before 规则**



## 2. volatile 介绍

- 作用 : 声明于变量 , 禁用CPU缓存. 读写均直接从内存中操作 . 

-  存在问题 : 

  ```java
  class VolatileExample {
    int x = 0;
      
    volatile boolean v = false;
      
    // 线程 A 执行 writer()
    public void writer() {
      x = 42;
      v = true;
    }
      
    // 线程 B 执行 reader()
    public void reader() {
      if (v == true) {
        // 这里 x 会是多少呢？
      }
    }
  }
  ```
  
  - Java 1.5 之前 ,  当执行到 `v == true` 时 ,   后续读取到 `x` 的值 , 可能还是 0  . 这是由于我们前面所说的过的CPU缓存导致的.   
  - 而 Java 1.5 以及之后的版本就不会出现这种情况了 , 读取到的必然是 前面赋值的 42 .  具体是原因就是 **Happens-Before 规则**



## 3. Happens-Before 六大规则

- 简单描述 : 前面的操作结果对后续的操作结果是可见的.

  > 个人总结 :  就是 `volatile` 变量 的**前面的变量**改动的结果 , 对 `volatile` 变量是可见的. 

### 3.1. 顺序性规则

- **`volatile` 修饰的变量相关的代码** 的前面的代码 , 是不会被指令重排到 其之后的.



### 3.2. 变量规则

- 一个 `volatile` 变量的写操作  , 对于后续的这个 `volatile` 变量的读操作是可见的 .  ( 也可以简单理解为禁用CPU缓存.)



### 3.3. 传递性

- 描述 : 如果 A Happens-Before B，且 B Happens-Before C，那么 A Happens-Before C。

  ![](http://assets.studymachine.cn/img/202111170026206.png)

-  简单描述 :

  - 线程A  `x = 42`  在  happen before `写变量 v = true`  (规则1)
  - 线程A `写变量 v = true`  happen before `读变量 v = true` (规则2)
  - 所以 , 由于传递性 `线程A 中的 x = 42` , 对 `线程B中 读取到 v = true`时 , 是可见的 .



### 3.4. 管程中锁的规则

- **描述** : 对一个锁的`解锁` **Happens-Before** 于后续对这个锁的`加锁`

- **什么是管程 ?** 

  - **管程**是一种通用的同步原语，在 Java 中指的就是 synchronized
  - synchronized 是 Java 里对管程的实现

- **个人理解** : 即加锁修改共享变量的操作 , 在解锁后 , 该修改被其他线程再次加锁时 , 是可见的 .  如下 : 

  ```java
  synchronized (this) { // 此处自动加锁
    // x 是共享变量, 初始值 =10
    if (this.x < 12) {
      this.x = 12; 
    }  
  } // 此处自动解锁
  ```

  线程A 加锁  设置 x = 12  , 然后解锁 .  线程B 再次加锁后进入同步代码块 , 是可以看到 x = 12的 . 





### 3.5. 线程start( ) 规则

- **描述** : 线程A中调用 线程B的  `start()` 方法 , 则 **线程A  happens-before 线程B**  方法 , 线程 A 中调用B的start() 方法之前的修改 , 对后续线程B 来说是可见的 . 

- 例如 : 在线程A 中运行如下代码 . 

  ```java
  Thread B = new Thread(()->{
    // 主线程调用 B.start() 之前
    // 所有对共享变量的修改，此处皆可见
    // 此例中，var==77
  });
  // 此处对共享变量 var 修改
  var = 77;
  // 主线程启动子线程
  B.start();
  ```

  B start() 后 , 在 B 中看到 var 的值应该是 77 . 







### 3.6. 线程 join ( ) 规则

- 描述 : 线程 A 中调用 B 的 `join()` 方法.  则说B 中任意操作 , 均 happens-before A的调用B 的 `join()` 的这个操作 .   

- 换句话说 : A中调用 B 的join , 则 B 中任何操作 , 对 A 中 join 之后来说 , 都是可见的.  如下代码所示 : 

  ```java
  Thread B = new Thread(()->{
    // 此处对共享变量 var 修改
    var = 66;
  });
  // 例如此处对共享变量修改，
  // 则这个修改结果对线程 B 可见
  // 主线程启动子线程
  B.start();
  B.join()
  // 子线程所有对共享变量的修改
  // 在主线程调用 B.join() 之后皆可见
  // 此例中，var==66
  ```

  



## 4. final 关键字

- **作用 :** 被修饰的变量是不可变的 , 被修饰的类是不可被继承的. 

- jdk1.5 之前 , 被 `final` 修饰的变量 , 会由于编译器优化 , 构造函数指令重排的原因 , 导致可以看到被final 修饰的值是会变化的

- jdk 1.5 之后 , 只要不出现 **构造函数逸出** 的现象 , 被final 修饰的变量 , 是遵循其语义的 . 

- 什么是 **构造函数逸出** ?

  - 构造中 , 指定 `this` 赋值给全局变量 .  

  ```java
  // 以下代码来源于【参考 1】
  final int x;
  // 错误的构造函数
  public FinalFieldExample() { 
    x = 3;
    y = 4;
    // 此处就是讲 this 逸出，
    global.obj = this;
  }
  ```