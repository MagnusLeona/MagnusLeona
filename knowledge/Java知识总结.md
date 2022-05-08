#### Java知识总结



##### Java容器

​	Java容器主要分为两大类： Collection和Map

- Collection
  - List
    - ArrayList
    - LinkedList
  - Set
    - HashSet
    - TreeSet
- Map
  - HashMap
    - LinkedHashMap
  - HashTable



​	Java并发容器：

- ConcurrentHashMap
  - Java7中采用分段锁的方式来减少锁的竞争，使用一个Segment表示多个Entry的方式，每次进来先锁住Segment中对应的锁。Java8中采用了细粒度的锁，直接锁住hash之后得到的Entry节点第一个值。同时，Java8之后，HashMap优化为单节点hash冲突次数超过8就自动转化为红黑树结构。
  - 默认容量是16。默认负载因子0.75。
  - <font color="red">为什么ConcurrentHashMap的Key和Value不能为null，但是HashMap却可以？</font>
    - 当我们通过get(key)的方式获取对应的value时，如果我们获取到的是null，那我们是没法判断，这个值是在put(key, value)的时候就为空，还是这个key从来没有做过映射。当我们首先从map中get某个key的时候，由于map中的key不存在，那么就会返回null。这之后如果我们使用contains来判断key是否存在，如果这时候有个并发线程给这个key写入了null值，则我们获取的key是存在的，这样就会产生二义性。
- CopyOnWriteArrayList
  - 并发版的ArrayList，当增加和删除元素的时候，会创建一个新数组，在新数组中增加或删除指定对象后，用新数组代替原来的数组。所以只适用于读多写少的场景，因为可能会读到脏数据。读的时候不会加锁，读取的是当前的副本。
- CopyOnWriteArraySet
- ConcurrentLinkedQueue
  - 使用CAS操作完成入队/出队的操作。
- ConcurrentLinkedDeque
  - 双向队列（基于双向链表）。支持FIFO和FILO。没法用size获取准确的大小。
- ConcurrentSkipListMap
- ConcurrentSkipListSet
- ArrayBlockingQueue
  - 阻塞队列，基于数组。初始化的时候必须指定大小。添加元素的时候，如果满了就一直阻塞直到有位置产生，但也支持直接返回和超时等待，通过ReentrantLock保证线程安全。
- LinkedBlockingQueue
  - 基于链表实现的阻塞队列。相比较于ConcurrentLinkedQueue，多了个容量限制，如果不设置，则为最大整数值。
- LinkedBlockingDeque
  - 基于双向链表实现的阻塞队列。有容量限制。
- PriorityBlockingQueue
- SynchronousQueue
- LinkedTransferQueue
- DelayQueue



##### HashMap相关知识

- HashMap实现

  - JDK 7之前，内部是通过数组+链表的方式实现。创建了一个Entry(Entry是一个Key-Value的节点)的数组，初始化大小为Capacity=16。

    ```java
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        final int hash;
    }
    ```

  - JDK8之后，内部通过数组+链表+红黑树的方式实现，当某个节点上哈希冲突的数量超过8之后，链表将转化为红黑树。

  - put方法被调用的时候，HashMap会根据key自动生成hashcode值，然后根据hashcode确定该Entry该放到数组的哪个位置。如果发现该位置下已经有Entry节点，则对要放入的key和当前存在的key做equals比较，如果相同，则直接覆盖。如果不同，表示遇到了hash碰撞，则将要放入的节点放到存在的Entry之后。

  - Hash冲突的解决方法：

    - 拉链法：遇到hash冲突，直接在冲突的节点加上链表，一直往后追加。
    - 开放定址法：也称为再散列法，基本思想是，当关键字key的hash发生冲突时，从冲突的位置往后一直查找空位置。
    - 再哈希法：使用多个哈希方法，当产生冲突时，一直调用不同的hash算法直到不再冲突。

- HashMap线程不安全的原理：

  - JDK1.7之前，HashMap在多线程并发情况下，扩容的时候可能会出现死循环和数据丢失的情况。原因是，对于有hash冲突的节点，冲突的节点出是一个单链表。扩容的时候需要对所有值进行重新hash，并采用头插法的方式去更新每一个hash节点。多线程同时做扩容操作的时候，会导致链表的节点插入位置异常，造成死循环和数据丢失。
  - JDK1.8后，并发情况下，HashMap进行put操作的时候，可能存在数据覆盖的情况。A线程在hash完成之后，正好要放到对应的数组中时，线程挂起，这时候另外一个线程进来在这个数组的相同位置设置一个值，这时候A继续执行就会覆盖掉这个值。JDK1.8之后 ，扩容采用了resize()的方法，使用的是尾插法，不会再出现JDK1.7的情况。

