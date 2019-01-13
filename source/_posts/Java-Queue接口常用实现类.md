---
title: Java Queue接口常用实现类
date: 2019-01-09 18:56:39
tags:
    - Collection
---
[Queue](https://docs.oracle.com/javase/8/docs/api/java/util/Queue.html)表示队列，在Java中，一般是FIFO（先进先出）的队列，但也有不是先进先出的队列，比如优先级队列

## 接口
```java
public interface Queue<E> extends Collection<E> {
    // 添加一个元素到队列，在先进先出队列中，是添加到队列尾部
    // 如果由于容量不足插入失败，则抛出异常，不会返回false
    boolean add(E e);
    
    // 和add方法一样，不过不同之处在于，这个方法添加失败一般是返回false，而不是抛出异常
    boolean offer(E e);
    
    // 移除队列尾部的元素（最后进来的元素），如果队列为空，则抛出异常，不返回false
    E remove();

    // 和remove方法一样，不过这个方法在队列为空时会返回false，不抛出异常
    E poll();

    // 查看队列头部的元素（最先进来的元素），如果队列为空，则抛出异常，不返回null
    E element();
    
    // 和element方法一样，不过这个在队列为空时，返回null，不抛出异常
    E peek();
}
```

队列也可以分为阻塞队列和非阻塞队列，则阻塞队列新增了几个方法
```java
    // 添加一个元素到队尾，队列已满的时候抛出异常
    boolean add(E e);

    // 添加一个元素到队尾，队列已满的时候返回false
    boolean offer(E e);

    // 插入一个元素到队尾，队列已满的时候则阻塞，直到成功插入元素
    void put(E e) throws InterruptedException;
    
    // offer(E e)的阻塞版，不过仅仅阻塞timeout的时间，超时后仍然无法插入元素则返回false
    boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException;
    
    // 获取并移除队首元素，如果队列为空则阻塞，直到成功取出元素
    E take() throws InterruptedException;
    
    // poll()的阻塞版本，不过仅仅阻塞timeout的时间，超时后仍然无法取出元素则返回false
    E poll(long timeout, TimeUnit unit)
        throws InterruptedException;

    // 返回队列中的剩余容量，如若队列是无边界的，则返回Integer.MAX_VALUE
    int remainingCapacity();

    // 移除队列中的特定元素，如果存在多个，则全部移除并返回true，元素不存在则返回false
    boolean remove(Object o);

    // 移除队列里面的所有元素，并存放到指定的Collectino中
    int drainTo(Collection<? super E> c);
    // 最多移除移除队列里面的maxElements个元素，并存放到指定的Collectino中
    int drainTo(Collection<? super E> c, int maxElements);
```
* __不阻塞__ 调用之后如果操作失败（一般都是因为队列容量不足或者队列为空），则立即返回，不发生阻塞，add(e)，offer(e)remove(e)，poll()，element()，peek()等，在`Queue`接口的方法一般都是不阻塞的

* __阻塞__ 调用之后如果暂时无法操作成功，则一直阻塞到操作成功，put(e)，take()等


* __限时阻塞__ 调用之后如果暂时无法操作成功，则阻塞一段时间，超时后仍为操作仍然未成功则返回，offer(e,timeout,unit)，poll(timeout,unit)等


## 非线程安全的实现类

### PriorityQueue

[PriorityQueue](https://docs.oracle.com/javase/8/docs/api/java/util/PriorityQueue.html)是优先级队列，它不是FIFO的，而是根据元素的优先级，优先级高的或者低的先出队列，它是非阻塞队列。不允许存放`null`元素

* 构造器
```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {

    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }

    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }

    @SuppressWarnings("unchecked")
    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }

    @SuppressWarnings("unchecked")
    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }

    @SuppressWarnings("unchecked")
    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }
}
```

* 默认值
默认的队列容量为11
```java
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
```

* 底层存储结构
底层存储结构为堆，也就是数组，它扩容策略如下：
```java
        int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                         (oldCapacity + 2) :
                                         (oldCapacity >> 1));
```
就是说如果旧容量小于64，则新容量变成旧容量×2 + 2，如果旧容量大于等于64，则新容量变成旧容量×1.5
```java
    /**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    transient Object[] queue; // non-private to simplify nested class access
```

非线程安全的`Queue`实现类就只有这一个（不包括`Deque`的实现类，`Deque`拓展了`Queue`接口），其他的大部分都是一些内部类

## 线程安全的实现类

### PriorityBlockingQueue

[PriorityBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/PriorityBlockingQueue.html)是无边界的阻塞队列，它和`PriorityQueue`一样能根据`Comparator`按照特定的顺序将元素出队列。不允许存放`null`元素

* 构造器
```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {

    public PriorityBlockingQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    public PriorityBlockingQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }

    public PriorityBlockingQueue(Collection<? extends E> c) {
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        boolean heapify = true; // true if not known to be in heap order
        boolean screen = true;  // true if must screen for nulls
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            heapify = false;
        }
        else if (c instanceof PriorityBlockingQueue<?>) {
            PriorityBlockingQueue<? extends E> pq =
                (PriorityBlockingQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            screen = false;
            if (pq.getClass() == PriorityBlockingQueue.class) // exact match
                heapify = false;
        }
        Object[] a = c.toArray();
        int n = a.length;
        // If c.toArray incorrectly doesn't return Object[], copy it.
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, n, Object[].class);
        if (screen && (n == 1 || this.comparator != null)) {
            for (int i = 0; i < n; ++i)
                if (a[i] == null)
                    throw new NullPointerException();
        }
        this.queue = a;
        this.size = n;
        if (heapify)
            heapify();
    }
}
```

* 默认值
默认容量为11（因为底层是数组，而这是一个无边界队列，所以随着元素的增加需要进行扩容操作）
```java
    /**
     * Default array capacity.
     */
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

* 底层存储结构
底层为数组，扩容策略和`PriorityQueue`是一样的
```java
    private transient Object[] queue;
```

### ArrayBlockingQueue

[ArrayBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ArrayBlockingQueue.html)是有边界（即有容量限制）的阻塞队列，基于数组实现。不允许存放`null`元素

* 构造器
```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);

        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }
}
```

* 默认值
无，队列容量的值必须在构造器指定

* 底层存储结构
底层是数组，容量指定之后就无法改变
```java
    /** The queued items */
    final Object[] items;
```

### LinkedBlockingQueue

[LinkedBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedBlockingQueue.html)是一个阻塞队列，它既可以是有边界的，也可以是无边界的。不允许存放`null`元素

* 构造器
```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {

    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }

    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
}
```

* 默认值
队列容量capacity默认为Integer.MAX_VALUE，也就是默认为无边界队列
```java
    /** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;
    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();
    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

* 底层存储结构
底层存储结构为链表
```java
    /**
     * Linked list node class
     */
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }

    /**
     * Head of linked list.
     * Invariant: head.item == null
     */
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     */
    private transient Node<E> last;    
```

### ConcurrentLinkedQueue

[ConcurrentLinkedQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentLinkedQueue.html)是无边界的阻塞队列，基于链表实现，使用了无锁算法提高并发性能。不允许存放`null`元素

* 构造器
```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {

    public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }

    public ConcurrentLinkedQueue(Collection<? extends E> c) {
        Node<E> h = null, t = null;
        for (E e : c) {
            checkNotNull(e);
            Node<E> newNode = new Node<E>(e);
            if (h == null)
                h = t = newNode;
            else {
                t.lazySetNext(newNode);
                t = newNode;
            }
        }
        if (h == null)
            h = t = new Node<E>(null);
        head = h;
        tail = t;
    }
}
```

* 默认值
无

* 底层存储结构
底层存储结构为链表
```java
    private transient volatile Node<E> head;

    private transient volatile Node<E> tail;

    private static class Node<E> {
        volatile E item;
        volatile Node<E> next;
    }    
```

### SynchronousQueue

[SynchronousQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/SynchronousQueue.html)是一个阻塞队列，它的特点是在插入元素的时候必须有另一个线程在等待取出元素，或者反过来，它没有容量的概念，因为这个队列是不存储元素的，每次将一个元素入队列的时候必须在另一个线程将其取出才能把下一个元素入队列。不允许存放`null`元素

* 构造器
```java
public class SynchronousQueue<E> extends AbstractQueue<E>
    implements BlockingQueue<E>, java.io.Serializable {

    public SynchronousQueue() {
        this(false);
    }

    /**
     * Creates a {@code SynchronousQueue} with the specified fairness policy.
     *
     * @param fair if true, waiting threads contend in FIFO order for
     *        access; otherwise the order is unspecified.
     */
    public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }

}
```

* 默认值
无

* 底层存储结构
底层存储结构为`Transferer`
```java
    private transient volatile Transferer<E> transferer;

    /** Dual stack */
    static final class TransferStack<E> extends Transferer<E> {
        /*
         * This extends Scherer-Scott dual stack algorithm, differing,
         * among other ways, by using "covering" nodes rather than
         * bit-marked pointers: Fulfilling operations push on marker
         * nodes (with FULFILLING bit set in mode) to reserve a spot
         * to match a waiting node.
         */

        /* Modes for SNodes, ORed together in node fields */
        /** Node represents an unfulfilled consumer */
        static final int REQUEST    = 0;
        /** Node represents an unfulfilled producer */
        static final int DATA       = 1;
        /** Node is fulfilling another unfulfilled DATA or REQUEST */
        static final int FULFILLING = 2;
        /** The head (top) of the stack */
        volatile SNode head;

        /** Node class for TransferStacks. */
        static final class SNode {
            volatile SNode next;        // next node in stack
            volatile SNode match;       // the node matched to this
            volatile Thread waiter;     // to control park/unpark
            Object item;                // data; or null for REQUESTs
            int mode;
        }
    }
```

### LinkedTransferQueue

[LinkedTransferQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedTransferQueue.html)是无边界的传输队列（[TransferQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/TransferQueue.html)。不允许存放`null`元素