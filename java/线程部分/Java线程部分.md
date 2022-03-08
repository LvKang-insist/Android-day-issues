### 如何停止一个线程

- 官方停止线程的方法被废弃了，所以不能直接简单的停止线程。

- 所以就需要设计一个可以随时被中断而取消的任务线程。

#### 为什么不能简单的停止一个线程

栗子：两个线程，线程1去访问一个带锁的方法去读取文件，然后获取到了锁，这个时候线程2也要去读写文件，但是只能被阻塞了。

此时如果停止了线程1，那么就会导致锁无法被释放，从而导致线程2一直在阻塞。甚至说如果线程比较多，就可能会出现死锁的问题。

需要解决的：

- Thread1 被停止后，必须立即释放锁
- Thread2 如果拿到了锁，但是内存状态异常。因为之前的 Thread 1没有读完文件就停止了。

#### 协作的任务执行模式

线程在设计的时候是和任务进行绑定的，任务执行完了，那么线程自然也就自己停止了。我们想让线程结束本质上就是让任务结束，而不是让线程结束。如果让任务结束了，那么线程自然有机会去释放锁等一系列的操作，自然也就不会引起其他问题了。所以我们需要：

- 通知目标线程自行结束，而不是强制停止

- 目标线程应当具备处理中断的能力

- 中断方式

  - Interrupt 的原生支持

    ```java
    //目标线程
    Thread thread = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
                //清理
            }
        }
    });
    ```

    ```java
    //中断宣传
    thread.interrupt();
    ```

    在有些情况下，中断线程也是不好使的，如下：

    ```java
    Thread thread = new Thread(new Runnable() {
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                System.out.println(i);
            }
        }
    });
    thread.start();
    ```

    ```java
    thread.interrupt();//中断线程
    ```

    这种情况下中断线程是没有用的，因为 Thread 不支持这种。如果要可以进行中断，就只能是如下写法了：

    ```dart
    class InterruptThread extends Thread {
         @Override
         public void run() {
             for (int i = 0; i < 10000; i++) {
                 System.out.println(i);
                 if (interrupted()) {
                     break;
                 }
             }
         }
     }
    ```

    每次去判断一下线程 有没有中断，如果中断了就结束任务，清理资源就行了

    interrupted() 与 isInterrupted 

    - interrupted() 是静态方法，获取当前线程的中断状态，并清空当前运行的线程到的状态
      - 中断状态调用后清空，重复调用后续返回false
    - isInterrupted() 是非静态方法，获取该线程的中断状态，不清空，只是一个简单的读取状态
      - 调用的线程对象对应的线程
      - 可重复调用，中断清空前一直返回true

  - boolean 标志位

    ```dart
    static class InterruptThread extends Thread {
        volatile boolean isStopped = false;
    
        @Override
        public void run() {
            for (int i = 0; i < 10000; i++) {
                System.out.println(i);
                if (isStopped) {
                    break;
                }
            }
        }
    }
    ```

    需要注意的是标志位必须要使用 volatile 关键字，否则就会存在可见性的问题。

  - 两者对比

    |                 | interrupt | boolean 标志位         |
    | --------------- | --------- | ---------------------- |
    | 系统方法(selep) | 是        | 否                     |
    | 使用 JNI        | 是        | 否                     |
    | 加锁            | 是        | 否                     |
    | 触发方式        | 抛异常    | 布尔值判断，也可抛异常 |

    - 如果需要支持使用系统方法时进行中断，就可以使用 interrupt。(功能性)]
    - 其他情况使用 boolean 标志位(性能)

#### 总结

- 为什么线程不应该被直接 stop

  因为涉及到线程执行完后的资源清理工作。

- 线程内置的中断机制的使用和原理

- 通过 volatile boolean 标志位通知线程停止



### 如何写出线程安全的程序

- 什么是线程安全
  多个线程同时操作一个可变的资源。因为每个线程都会将资源拷贝一份到自己的线程内存中，修改之后再同步到主内存中，如果多个线程同时对一个资源进行操作的话，就会出现线程安全的问题。

- 如何实现线程安全

  - 不共享资源

    - 可重入函数

      ```java
      public static int addTwo(int num){
          return num+2;
      }
      ```

      可重入函数，该函数会返回一个值出去，但是中间不会涉及到外部内存的访问，所以他是线程安全的。

    - ThreadLocal：

      会将值存入到对应的 Thread 类中，每次修改或者添加都是针对当前线程的，所以不会有线程安全问题。

      与 WeakHashMap 进行比较：

      |           | ThreadLocalMap                       | WeakHashMap                                    |
      | --------- | ------------------------------------ | ---------------------------------------------- |
      | 对象持有  | 弱引用                               | 弱引用                                         |
      | 对象 GC   | 不影响                               | 不影响                                         |
      | 引用清除  | 1，主动清除，<br />2，线程退出时清除 | 1，主动清除<br />2，GC 后移除 (ReferenceQueue) |
      | Hash 冲突 | 开放定址法                           | 单链表法                                       |
      | Hash 计算 | 神奇的数字倍数                       | 对象 hashCode 再散列                           |
      | 使用场景  | 对象较少                             | 通用                                           |

      使用建议：

      1. 声明为全局静态 final 成员

         应为在 setValue 的时候是以 ThreadLocal 为 key 的，如果每次都创建新的对象就会导致之前存入的数据全部拿不出来。

      2. 避免存入大量资源

      3. 使用完成后及时移除对象

    

  - 共享不可变资源

  - 共享可变资源

    - 可见性

      对资源进行修改之后需要让别的线程都知道资源已经发生了改变。

      1. 使用 final
      2. 使用 volatile 关键字
      3. 加锁，锁释放的时候会强制将缓存刷入到主内存。

    - 操作原子性

      对资源操作的时候如果不确保原子性，可能在操作资源的时候其实值已经被修改了。

      1. 加锁，保证操作的互斥性
      2. 使用 CAS 
      3. 使用原子数值类型如 (AtomicInteger)

    - 禁止重排序

      cup 在执行的时候不是一行一行执行的，而是可能执行好几行，执行完后会将结果放入到缓存中，等到需要的时候在拿出来。如果在此期间资源被修改的话，就会造成之前计算的结果不准确，所以需要禁止重排序。

      禁止重排序可以通过，final 或者 volatile 来声明。

      例如：单例中的双层验证锁：

      ```java
      class Manager{
      	private volatile static Manager single;
          
          public static Manager getInstance(){
              if(single == null){ //1
                  synchronized(Manager.class){
                      if(single == null){
                          single = new Manager();//2
                      }
                  }
              }
              return single;
          }
      }
      ```

      如果没有使用 volatile ，在调用 getInstance 中的第 2 处，将对象赋值给了 single。此时第二个线程来了，在第一处判断已经不为空了，所以直接拿去使用了。但是有可能会出现的问题就是虽然对象已经不为空了，但是该对象可能还没有初始化完成，所以就有可能出现问题。

      

### ConcurrentHashMap 如何支持并发访问

- JDK 1.5：分段锁，必要时加锁

  对 key 进行 hash 运算，然后用高位去找到对应的 `segment` ，用低位去找 `segment` 对应的 `table `。每一个 `segment` 都会被加锁。相比与 `HashTable` 的全部加锁，给其中的每一段加锁显然是好了不少。

  这里需要注意的是对 key 进行 hash 之后的值一定要均匀，否则的话大多数的 key 都指向了同一个 `segment`，这也就和 `HashTab` 没有太大区别了。

  但是在 1.5 的时候，如果 key 是一个整数，那么这个 hash 值的高位对于 三万多以下的整数得到的结果都是最后一个 `segment`。就算是高于 3万，得到的结果都是非常不均匀的。这算是这个版本的一个缺陷。

- JDK 1.6：优化 二次 Hash 算法

  JDK 1.5 的 hash 算法：

  ```java
  static int hash(Object x){
      int h = x.hashCode();
      h += ~(h << 9);
      h ^= (h >>> 14);
      h += (h << 4);
      h ^= (h >>> 10);
  	return h;
  }
  ```

  如果传入的是三万多以下的整数，最后计算出来 hash 的高位始终是 15，这样的话就没办法均匀的分布在 `segment` 中了。和 `HashTab` 就差不多了。

  JDK 1.6 的 hash 算法：

  ```java
  static int hash(int h){
      h += (h << 15) ^ 0xfffffcd7d;
      h ^= (h >>> 10);
      h += (h << 3);
      h ^= (h >>> 6);
      h += (h << 10) + (h << 14);
  	return h ^ (h >>> 16);
  }
  ```

  在 1.6 里面修改了 hash 的方法，使得最后计算的高位可以均匀的分布。

- JDK 1.7：段懒加载，volatile & cas

  在之前的版本中，只要将 `ConcurrentHashMap` new 出来之后，16 个`segment` 都会被创建出来。

  在 1.7 中，不会直接创建 `segment` 了，使用了懒加载，使用哪个，才会将哪个创建出来。

   但是这也有一个问题，如果线程 A 访问了第一个 `segment` 并且初始化了，此时的线程2可能由于可见性的问题访问不到这个刚刚初始化的 `segment` 。

  正是这个原因，在1.7 里面大量的使用了 `getObjectVolatile()` 方法来实现对 `segment` 的 volatile 访问。

- JDK 1.8：摒弃段，基于 HashMap 原理的并发实现

  在 1.8 中，直接取消了 `segment`，通过 `getObjectVolatile()`去访问 table 了。然后对 table 的元素进行了加锁。

  还有就是将之前的 table数组 + 单向链表的数据结构变成了 table 数组 +单向链表 + 红黑树了。

#### CHM 基础知识

- 默认创建 16个 `Segment` 对象的数组(1.6-1.7)，1.8之后采用了懒加载。每个 Segment 对应一个 table，每个table 里面是一个链表
- 对于 table 长度大于8 的链表，就会采用红黑树，可以改进性能(在 1.8 中)

#### CHM 如何计数

- JDK 5~7 基于段元素的个数进行求和，二次不同就加锁在求一次
- JDK 1.8 引入了 CounterCell ,本质上也是分段计数

#### CHM 是弱一致性的

- 添加元素后不一定能马上读到，因为在添加的时候可能已经读完了。
- 清空之后可能仍然有元素，有可能还没清除完，又添加上了。
- 遍历之前的段元素的变化会读到。例如刚遍历到段 14，此时在段15中添加了元素，这个时候就会遍历到
- 遍历之后的段元素变化读不到
- 遍历时元素发生变化不抛异常

#### HashTable 的问题

| HashTable                        | ConcurrentHashMap                                       |
| -------------------------------- | ------------------------------------------------------- |
| 对 HashTable 对象加锁            | 小锁，分段锁(JDK 5-7)，桶节点锁(JDK 8)                  |
| 直接对方法进行加锁               | 先尝试获取，失败了在加锁，不一定直接加锁                |
| 读写锁共用只有一把锁，从头锁到尾 | 分离读写锁：读失败再加锁(5-7)。volatile 读，CAS 写(8)。 |

#### 如何进行锁优化

- 长锁不如短锁：尽可能锁住必要的部分
- 大锁不如小锁：尽可能对加锁对象进行拆分
- 公锁不如私锁：尽可能将锁逻辑放在私有代码中
- 嵌套锁不如扁平锁：尽可能避免使用嵌套锁
- 分离读写锁：尽可能读写分离
- 粗话高频锁：尽可能合并处理频繁过短的锁
- 消除无用锁：可能能不加锁，或使用 volatile 代替锁

