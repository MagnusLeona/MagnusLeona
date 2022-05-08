## JUC 核心知识



### Synchronized关键字

- synchronized关键字，锁对象的时候，synchronized(this)和synchronized方法都是锁当前对象。

- synchronized关键字锁静态同步方法的时候，所得是当前类型的类对象，即当前的class。

- synchronized方法，加锁的目的是保证操作的原子性。

- synchronized关键字，同步方法只会影响锁定同一个对象的同步方法，不会影响其他线程对非同步方法的调用，或者锁其他资源的同步方法。

- synchronized关键字，只能保证当前方法的原子性，不能保证多个业务方法的互相访问的原子性。在商业开发中，多方法要求结果访问的原子性的场景，需要多个方法都加锁，且锁定同一个资源。（一般来说，商业项目中不考虑业务逻辑上的脏读问题）

- synchronized属于可重入锁，同一个线程多次调用同步代码，锁定的如果是同一个对象，则可以重入。

- synchronized关键字中，子类方法覆盖父类方法，在子类方法里面可以调用父类的同步方法，相当于锁的重入。

- synchronized关键字的同步方法中如果发生异常，将自动释放锁资源，不会影响其他线程的执行。（注意：同步业务逻辑中，如果发生异常要怎么处理）

- synchronized关键字，尽量在商业开发中避免使用同步方法，锁的粒度要精细，使用同步代码块，仅锁重要的代码块。细粒度解决同步问题，可以提高效率。

- synchronized关键字，同步代码块一旦变更，会有一个临时的锁引用执行锁对象，和真实的引用没有直接关联。在锁未释放之前，修改锁对象的引用，不会影响同步代码块的执行。

- synchronized关键字，在定义同步代码块的时候，不要使用常量对象作为锁对象。

- synchronized关键字锁的对象，可以通过wait方法手动释放锁，这个时候线程会重新回到锁对象的等待队列之中，直到被notify通知或者中断为止。notify方法，可以唤醒处于等待中的线程（随机挑选一个线程唤醒）。notifyAll方法会唤醒所有处于等待队列中的线程。线程调用sleep方法，会交出CPU资源，但是不会释放锁。 

- synchronized关键字，属于重量级锁，不公平锁。在进行锁的获取/释放的时候涉及线程的阻塞和唤醒，会进行线程的用户态和内核态的切换，造成线程上下文切换的资源浪费。

- synchronized关键字原理：

  - synchronized修饰代码块的时候，JVM会在代码块添加`monitorenter`和`monitorexit`指令来标记方法需要同步。

  - synchronized修饰同步方法的时候，JVM会给方法添加`ACC_SYNCHRONIZED`命令来标记同步方法。

  - `monitorenter`、`monitorexit`和`ACC_SYNCHRONIZED`都是基于`Monitor`实现的。

  - `Monitor`是一个同步工具，也是一种同步机制。

  - 实例对象中的对象头中有Mark Word，Mark Word中会有指向`monitor`的指针（指向重量级锁）。

  - 在Hotspot虚拟机中，`Monitor`是由`ObjectMonitor`实现的。

    ```c++
    ObjectMonitor() {
        _header = NULL; 
        _count = 0;     // 记录线程获取锁的次数
        _waiters = 0;   
        _recursions = 0;  // 锁的重入次数
        _object = NULL;
        _owner = NULL;    // 指向持有ObjectMonitor对象的线程
        _WaitSet = NULL;   // 处于wait状态的线程，会被加入到_WaitSet
        _WaitSetLock = 0;
        _Responsible = NULL;
        _succ = NULL;
        _cxq = NULL;
        FreeNext = NULL;
       	_EntryList = NULL;    // 处于等待锁block状态的线程，会被加入到_EntryList
        _SpinFreq = 0;
        _SpinClock = 0;
        _OwnerIsThread = 0;
    }
    ```

- 几种锁状态

  - 偏向锁：当锁对象第一次被线程获取时，虚拟机会把对象头中的锁标志位设置为01，并把偏向模式设置为1，表示进入偏向模式，并使用CAS操作把获取到这个锁的线程ID记录到对象头的`Mark Word`中。如果CAS操作成功，则持有偏向锁的线程每次进入这个锁相关的同步代码块的时候都不需要再进行任何同步操作。一旦有另外一个线程尝试获取这个锁的情况，偏向模式立刻结束，根据锁对象当前是否处于被锁定状态决定是否撤销偏向，撤销后标志位恢复到未锁定（01）或者轻量级锁定（00）的状态，后续的同步操作就需要按照轻量级锁的方式去执行。
  - 轻量级锁：在代码进入同步代码块的时候，如果当前同步对象没有被锁定（锁标志位为01），虚拟机首先会在当前线程栈中建立一个名为锁记录（Lock Record）的空间，用来存储锁对象当前的`Mark Word`的拷贝。然后，虚拟机将使用CAS操作尝试将对象的`Mark Word`更新为指向Lock Record的指针。如果更新成功，则表示成功获取到这个对象的锁，并且将对象头`Mark Word`中的锁标志位设置为00，表示此对象处于轻量级锁的状态。如果更新失败了，则表示至少有一个线程和当前线程竞争获取该对象的锁定状态。虚拟机会首先检查对象头的`Mark Word`是否指向当前线程的栈帧，如果是，说明当前线程已经拥有这个对象的锁，那直接进入同步代码块执行即可，否则说明这个锁对象已经被其他线程抢占了。如果出现两个以上的线程争抢同一个锁的情况，则轻量级锁需要膨胀为重量级锁，锁的标志位状态将变为10，此时`Mark Word`中将存储指向重量级锁的指针（ObjectMonitor）。轻量级锁相对于自旋锁，需要不断的进行CAS操作去判断锁状态，所以需要消耗一定的CPU资源。
  - 自旋锁：在轻量级锁膨胀为重量级锁的过程中，需要通过CAS的方式进行加锁、获取锁对象。
  - synchronized关键字在JDK6之后不断被优化为多种锁状态的组合。

- synchronized和ReentrantLock的区别：

  - synchronized是Java语言的关键字，基于JVM实现的，ReentrantLock是基于JDK提供的API实现的(lock/unlock配合使用)。
  - synchronized性能和ReentrantLock持平。
  - ReentrantLock多了一些其他特性，包括等待可中断，公平锁，锁绑定多个条件（Condition）。



### Volatile关键字

- volatile关键字，通知OS操作系统底层，在每次CPU计算的时候，都检查内存中的数据有效性，保证内存中最新的值被使用。普通变量使用的时候是直接放到寄存器中的。

- volatile关键字只能保证可见性，不能保证原子性。

- volatile第二个作用是禁止指令重排。

- atomic类型的数据，如 AtomicInteger,AtomicLong，等，属于同步类型，其中的每个方法都是原子性操作，可以保证线程安全。

- CountDownLatch门闩，可以和锁混合使用，或者替代锁，在门闩未完全放开之前等待，在门闩完全放开后执行。可以避免锁执行效率低下的问题。

- Java中锁的级别：偏向锁、轻量级锁、自旋锁、重量级锁。轻量级锁和自旋锁都是基于CAS(Compare And Swap)原理，该方法是由CPU内核提供的指令。自旋锁实现是在线程的用户态，会不断的尝试自旋获取锁，不会涉及到用户态和内核态的切换，所以不会有线程上下文切换的资源浪费。

- ReentrantLock可重入锁，继承自Lock,需要进行手动加锁解锁(必须要在finally里面进行解锁，避免出现死锁)。如果加锁没有成功就执行解锁会报错，所以lock.lock()方法不要放到try里面，避免加锁失败还要进行解锁的情况。同样的，一次加锁和多次释放锁也会报错 。

- ReentrantLock默认为非公平锁，可以在创建锁对象的时候传入true,声明为公平锁。默认为非公平获取锁的原因是CAS操作中，新来的线程会优先执行CAS操作，如果成功了就能占用锁，而在锁等待队列中的线程就会拿不到需要的锁。

  ```java
  public class FairReentrantLock {
      public static ReentrantLock lock = new ReentrantLock(true);
  
      public static void main(String[] args) {
          Thread[] threads = new Thread[5];
          Runnable runnable = () -> {
              lock.lock();
              try {
                  System.out.println(Thread.currentThread().getName() + " executed");
                  Thread.sleep(1000);
              } catch (InterruptedException e) {
                  e.printStackTrace();
              } finally {
                  lock.unlock();
              }
          };
  
          for (int i = 0; i < 5; i++) {
              threads[i] = new Thread(runnable, String.format("T %s", i ));
          }
  		// thread 1-5依次获取锁。
          for (Thread thread : threads) {
              thread.start();
          }
      }
  }
  ```

- Semaphore信号量，semaphore可以用于限流，初始化的信号量的时候会设置信号量数量，每次只有固定这个数量的线程可以同时执行同步代码块，其余代码只能等信号量释放之后去竞争信号量。

  ```java
  public class SemaphoreDemo {
      static Semaphore semaphore = new Semaphore(2);
  
      public static void main(String[] args) {
          Runnable runnable = ()-> {
              try {
                  if(semaphore.availablePermits() == 0) {
                      System.out.println("blocked");
                  }
                  semaphore.acquire();
                  System.out.println("acquired");
                  Thread.sleep(1000);
                  System.out.println("released");
                  semaphore.release();
              } catch (InterruptedException e) {
                  e.printStackTrace();
              }
          };
          ExecutorService executorService = Executors.newFixedThreadPool(5);
          executorService.execute(runnable);
          executorService.execute(runnable);
          executorService.execute(runnable);
          executorService.execute(runnable);
          executorService.execute(runnable);
          executorService.shutdown();
      }
  }
  ```

- CyclicBarrier循环栅栏，在初始化的时候会传入一个barrier大小数量n以及一个可执行Runnable，当有n个线程同时一起调用了await方法的时候，会执行Runnable的方法。await方法表示线程抵达栅栏，当所有线程抵达栅栏后，执行初始化的Runnable方法。CyclicBarrier是可循环使用的，只需要使用reset方法重置一下即可。

  ```java
  public class CyclicBarrierDemo {
      static CyclicBarrier cyclicBarrier = new CyclicBarrier(3, ()-> {
          System.out.println("jobs done");
      });
  
      public static void main(String[] args) {
          Runnable r = () -> {
              try {
                  Thread.sleep(1000);
                  System.out.println(String.format("%s reachs the barrier", Thread.currentThread().getName()));
                  cyclicBarrier.await();
              } catch (InterruptedException e) {
                  e.printStackTrace();
              } catch (BrokenBarrierException e) {
                  e.printStackTrace();
              }
          };
  
          ExecutorService executorService = Executors.newFixedThreadPool(3);
          executorService.execute(r);
          executorService.execute(r);
          executorService.execute(r);
          executorService.shutdown();
      }
  }
  ```

- CompletableFuture异步任务创建工具，使用了默认线程池是ForkJoinPool.commonPool。可以用于创建异步任务、任务异步回调、多个任务组合处理等工作。

  - 创建异步任务
    - supplyAsync执行CompletableFuture任务，支持返回值。
    - runAsync执行CompletableFuture任务，没有返回值。
  - 任务异步回调
    - thenRun
    - thenRunAsync



### 线程池

- 线程池的本质是使用ThreadPoolExecutor类，构造参数如下：

  - corePoolSize：核心线程数量
  - maximumPoolSize：最大线程数量
  - keepAliveTime：超过corePoolSize的线程数量的空闲线程可以维持多长时间
  - TimeUnit：时间单位
  - workQueue：阻塞队列，线程池已经满了的话就放到阻塞队列中等待线程池释放资源
  - ThreadFactory：线程工厂
  - RejectedExecutionHandler：拒绝策略
    - ThreadPoolExecutor.AbortPolicy：丢弃并抛错
    - ThreadPoolExecutor.DiscardPolicy：直接丢弃，不抛错
    - ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列中较老的等待任务。
    - ThreadPoolExecutor.CallerRunsPolicy：当前提交的线程直接运行。

- 线程池执行策略：

  - 当线程池中小于corePoolSize线程在执行时，新提交的任务直接新建线程执行。
  - 当线程池数量达到corePoolSize时，新提交的任务将放到阻塞队列workQueue。
  - 当阻塞队列也满了的时候，如果maximunPoolSize>corePoolSize，则新建线程执行。
  - 当提交的任务大于maximunPoolSize + workQueue时，直接采用RejectedExecutionHandler的处理策略。
  - 如果线程池中超过corePoolSize的线程处于空闲时，经过keepAlive的时间后将被销毁。

- 一般不推荐使用Executors的方式去创建线程池，因为这时候创建出来的线程池阻塞队列是无界的，可能会造成内存溢出。

  ```java
  public class MagnusExecutor {
  
      public static void main(String[] args) {
          // 阻塞队列都需要设置容量
          ArrayBlockingQueue arrayBlockingQueue = new ArrayBlockingQueue(100);
          ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                  5,
                  10,
                  10,
                  TimeUnit.SECONDS,
                  arrayBlockingQueue,
              	Executors.defaultThreadFactory(),
                  new ThreadPoolExecutor.AbortPolicy());
  
          for (int i = 0; i < 10; i++) {
              threadPoolExecutor.submit(() -> {
                  try {
                      Thread.sleep(5000);
                      System.out.println("Runnable:  " + Thread.currentThread().getName());
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              });
          }
          threadPoolExecutor.shutdown();
      }
  }
  ```

  