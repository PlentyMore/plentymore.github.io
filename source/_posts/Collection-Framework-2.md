---
title: Collection-Framework(2)
date: 2018-09-12 00:29:50
header-img: https://i.imgur.com/RAwKwAj.jpg
cdn: 'header-off'
tags:
    - Container
    - Collection
---

{% meting "850673" "netease" "song" %}
这部分内容本来是放在Collection-Framework(1)里面的，但是内容太多导致生成html后，部署到github pages发现部分内容丢失，只能拆开分成几个部分了。

# PriorityQueue
无边界的优先级队列的实现，队列里面的元素按照自然顺序顺序或者用户自定义的Comparator规则进行排序。
它不允许插入null元素，不是线程安全的。

## 成员变量
```java
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * Priority queue represented as a balanced binary heap: the two
     * children of queue[n] are queue[2*n+1] and queue[2*(n+1)].  The
     * priority queue is ordered by comparator, or by the elements'
     * natural ordering, if comparator is null: For each node n in the
     * heap and each descendant d of n, n <= d.  The element with the
     * lowest value is in queue[0], assuming the queue is nonempty.
     */
    transient Object[] queue; // non-private to simplify nested class access

    /**
     * The number of elements in the priority queue.
     */
    private int size = 0;

    /**
     * The comparator, or null if priority queue uses elements'
     * natural ordering.
     */
    private final Comparator<? super E> comparator;

    /**
     * The number of times this priority queue has been
     * <i>structurally modified</i>.  See AbstractList for gory details.
     */
    transient int modCount = 0; // non-private to simplify nested class access
    
    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;    
```

## 构造函数
```java
    /**
     * Creates a {@code PriorityQueue} with the default initial
     * capacity (11) that orders its elements according to their
     * {@linkplain Comparable natural ordering}.
     */
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    /**
     * Creates a {@code PriorityQueue} with the specified initial
     * capacity that orders its elements according to their
     * {@linkplain Comparable natural ordering}.
     *
     * @param initialCapacity the initial capacity for this priority queue
     * @throws IllegalArgumentException if {@code initialCapacity} is less
     *         than 1
     */
    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    /**
     * Creates a {@code PriorityQueue} with the default initial capacity and
     * whose elements are ordered according to the specified comparator.
     *
     * @param  comparator the comparator that will be used to order this
     *         priority queue.  If {@code null}, the {@linkplain Comparable
     *         natural ordering} of the elements will be used.
     * @since 1.8
     */
    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }

    /**
     * Creates a {@code PriorityQueue} with the specified initial capacity
     * that orders its elements according to the specified comparator.
     *
     * @param  initialCapacity the initial capacity for this priority queue
     * @param  comparator the comparator that will be used to order this
     *         priority queue.  If {@code null}, the {@linkplain Comparable
     *         natural ordering} of the elements will be used.
     * @throws IllegalArgumentException if {@code initialCapacity} is
     *         less than 1
     */
    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }

    /**
     * Creates a {@code PriorityQueue} containing the elements in the
     * specified collection.  If the specified collection is an instance of
     * a {@link SortedSet} or is another {@code PriorityQueue}, this
     * priority queue will be ordered according to the same ordering.
     * Otherwise, this priority queue will be ordered according to the
     * {@linkplain Comparable natural ordering} of its elements.
     *
     * @param  c the collection whose elements are to be placed
     *         into this priority queue
     * @throws ClassCastException if elements of the specified collection
     *         cannot be compared to one another according to the priority
     *         queue's ordering
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
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

    /**
     * Creates a {@code PriorityQueue} containing the elements in the
     * specified priority queue.  This priority queue will be
     * ordered according to the same ordering as the given priority
     * queue.
     *
     * @param  c the priority queue whose elements are to be placed
     *         into this priority queue
     * @throws ClassCastException if elements of {@code c} cannot be
     *         compared to one another according to {@code c}'s
     *         ordering
     * @throws NullPointerException if the specified priority queue or any
     *         of its elements are null
     */
    @SuppressWarnings("unchecked")
    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }

    /**
     * Creates a {@code PriorityQueue} containing the elements in the
     * specified sorted set.   This priority queue will be ordered
     * according to the same ordering as the given sorted set.
     *
     * @param  c the sorted set whose elements are to be placed
     *         into this priority queue
     * @throws ClassCastException if elements of the specified sorted
     *         set cannot be compared to one another according to the
     *         sorted set's ordering
     * @throws NullPointerException if the specified sorted set or any
     *         of its elements are null
     */
    @SuppressWarnings("unchecked")
    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }
```

## 常用方法
* public boolean add(E e)

* public boolean offer(E e) 
往队尾插入一个元素

* public E peek()
获取队头的元素

* public boolean remove(Object o)

* public boolean contains(Object o)

* public Iterator<E> iterator()

* public void clear()

* public E poll() 
移除队头的元素（队列是FIFO）

* public Comparator<? super E> comparator()

# ArrayBlockingQueue
有边界的阻塞队列，底层是数组实现的，该类位于concurrent包下，是线程安全的，不允许插入null元素。因为它是有边界的，
因此在元素填满队列之后再往里面添加元素将会发生阻塞，直到有空余的位置可以插入元素，往空队列取出元素的时候同理。

## 成员变量
```java
    /** The queued items */
    final Object[] items;

    /** items index for next take, poll, peek or remove */
    int takeIndex;

    /** items index for next put, offer, or add */
    int putIndex;

    /** Number of elements in the queue */
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;

    /**
     * Shared state for currently active iterators, or null if there
     * are known not to be any.  Allows queue operations to update
     * iterator state.
     */
    transient Itrs itrs = null;
```

## 构造函数
```java

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and default access policy.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and the specified access policy.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity, the specified access policy and initially containing the
     * elements of the given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @param c the collection of elements to initially contain
     * @throws IllegalArgumentException if {@code capacity} is less than
     *         {@code c.size()}, or less than 1.
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
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
```

## 常用方法
* public boolean add(E e)
往队尾插入一个元素，如果队列已满则抛出IllegalStateException

* public boolean offer(E e)
往队尾插入一个元素，如果队列已满则立即返回false

* public boolean offer(E e, long timeout, TimeUnit unit)

* public void put(E e) 
往队尾插入一个元素，如果队列已满则发生阻塞直到成功插入元素

* public E poll()
取出队头元素，如果队列为空则立即返回null

* public E poll(long timeout, TimeUnit unit)

* public E take()
取出队头元素，如果队列为空则发生阻塞一直到成功取出元素为止

* public E peek()
获取队头元素（不移除），队列为空则返回null

* public int size()

* public int remainingCapacity()

* public boolean remove(Object o)

* public boolean contains(Object o)

* public void clear()

* public Iterator<E> iterator()

# LinkedBlockingQueue
可自定义边界范围的阻塞队列，底层基于链表实现，是线程安全的，不允许存放null

## 成员变量
```java
    /** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;

    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

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

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

## 构造函数
```java

    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}, initially containing the elements of the
     * given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
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

```
 
 ## 常用方法
* public int size()
 
* public int remainingCapacity()
 
* public void put(E e)
往队尾添加一个元素，如果队列已满则发生阻塞直到成功存储元素

* public boolean offer(E e, long timeout, TimeUnit unit)
往队尾添加一个元素，队列已满则等待一段时间（timeout），超时后没能成功添加元素则返回false

* public boolean offer(E e)
往队尾添加一个元素，失败立即返回false，不发生阻塞

* public E take()
取出队头元素（移除），如果队列为空则发生阻塞直到成功取出元素

* public E poll()
取出队头元素（移除），成功则返回该元素，不成功返回null，不发生阻塞

* public E poll(long timeout, TimeUnit unit)
取出队头元素（移除），队列为空则等待一段时间（timeout），超时后仍未成功取出元素则返回null

* public E peek()
获取队头元素（不移除），队列为空则返回null，不发生阻塞

* public boolean remove(Object o) 
移除指定元素，成功返回true，不成功返回false，不发生阻塞

* public boolean contains(Object o)

* public void clear()

* public Iterator<E> iterator()

# ConcurrentLinkedQueue
无边界的非阻塞队列，底层基于链表，线程安全，不允许存放null元素。

## 成员变量
```java

    /**
     * A node from which the first live (non-deleted) node (if any)
     * can be reached in O(1) time.
     * Invariants:
     * - all live nodes are reachable from head via succ()
     * - head != null
     * - (tmp = head).next != tmp || tmp != head
     * Non-invariants:
     * - head.item may or may not be null.
     * - it is permitted for tail to lag behind head, that is, for tail
     *   to not be reachable from head!
     */
    private transient volatile Node<E> head;

    /**
     * A node from which the last node on list (that is, the unique
     * node with node.next == null) can be reached in O(1) time.
     * Invariants:
     * - the last node is always reachable from tail via succ()
     * - tail != null
     * Non-invariants:
     * - tail.item may or may not be null.
     * - it is permitted for tail to lag behind head, that is, for tail
     *   to not be reachable from head!
     * - tail.next may or may not be self-pointing to tail.
     */
    private transient volatile Node<E> tail;
```

## 构造函数
```java

    /**
     * Creates a {@code ConcurrentLinkedQueue} that is initially empty.
     */
    public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }

    /**
     * Creates a {@code ConcurrentLinkedQueue}
     * initially containing the elements of the given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
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
```

## 常用方法
* public boolean add(E e)
往队尾插入一个元素，因为这是一个无边界的队列，因此该方法永远不会抛出IllegalStateException或者返回false

* public boolean offer(E e)
往队尾插入一个元素，和上面的方法一样，该方法能返回ture或者抛出空指针异常

* public E poll()
取出队头元素（移除），不发生阻塞

* public E peek()
获取队头元素（不移除），不发生阻塞

* public boolean isEmpty()

* public int size()
如果队列中的元素数量大于Integer.MAX_VALUE，则返回Integer.MAX_VALUE，该方法的时间复杂度不为O(1)，而是 O(n)

* public boolean contains(Object o)

* public boolean remove(Object o)
移除元素，不发生阻塞

* public boolean addAll(Collection<? extends E> c)
试图将添加自己的元素到队列中将抛出IllegalArgumentException，比如`queue.add(queue)`

* public Iterator<E> iterator()

# Queue总结
* 队列是FIFO（先进先出）的
* 有阻塞队列和非阻塞队列，阻塞队列的阻塞是基于Condition实现的
* peek()只获取队头元素，而不移除元素，而且不发生阻塞
* 在阻塞队列中，poll()和offer()均不发生阻塞，take()可能发生阻塞 

# ArrayDeque
双向队列，底层基于数组实现，大小可以动态变化，因此没有容量限制，不是线程安全的，

## 成员变量
```java
    /**
     * The array in which the elements of the deque are stored.
     * The capacity of the deque is the length of this array, which is
     * always a power of two. The array is never allowed to become
     * full, except transiently within an addX method where it is
     * resized (see doubleCapacity) immediately upon becoming full,
     * thus avoiding head and tail wrapping around to equal each
     * other.  We also guarantee that all array cells not holding
     * deque elements are always null.
     */
    transient Object[] elements; // non-private to simplify nested class access

    /**
     * The index of the element at the head of the deque (which is the
     * element that would be removed by remove() or pop()); or an
     * arbitrary number equal to tail if the deque is empty.
     */
    transient int head;

    /**
     * The index at which the next element would be added to the tail
     * of the deque (via addLast(E), add(E), or push(E)).
     */
    transient int tail;

    /**
     * The minimum capacity that we'll use for a newly created deque.
     * Must be a power of 2.
     */
    private static final int MIN_INITIAL_CAPACITY = 8;

```

## 构造函数
```java

    /**
     * Constructs an empty array deque with an initial capacity
     * sufficient to hold 16 elements.
     */
    public ArrayDeque() {
        elements = new Object[16];
    }

    /**
     * Constructs an empty array deque with an initial capacity
     * sufficient to hold the specified number of elements.
     *
     * @param numElements  lower bound on initial capacity of the deque
     */
    public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }

    /**
     * Constructs a deque containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.  (The first element returned by the collection's
     * iterator becomes the first element, or <i>front</i> of the
     * deque.)
     *
     * @param c the collection whose elements are to be placed into the deque
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayDeque(Collection<? extends E> c) {
        allocateElements(c.size());
        addAll(c);
    }
```

## 常用方法
* public void addFirst(E e)
添加元素到队头，没有返回值

* public void addLast(E e)
添加元素到队尾

* public boolean offerFirst(E e)
插入元素到队头，只返回true或者抛出异常

* public boolean offerLast(E e)
插入元素到队尾，只返回true或者抛出异常

* public E removeFirst()
移除队头元素，没有元素则抛出异常

* public E removeLast()
移除队尾元素，没有元素则抛出异常

* public E pollFirst()
移除队头元素，没有元素则返回null

* public E pollLast() 
移除队尾元素，没有元素则返回null

* public E getFirst() 
获取队头元素（不移除），不存在则抛出异常

* public E getLast()
获取队尾元素（不移除），不存在则抛出异常
 
* public E peekFirst()
获取队头元素（不移除），不存在则返回null
 
* public E peekLast()
获取队尾元素（不移除），不存在则返回null

* public boolean removeFirstOccurrence(Object o) 
移除第一个出现的特定队列元素（从队头到队尾遍历），元素不存在返回false

* public boolean removeLastOccurrence(Object o)
 移除最后一个出现的特定队列元素（从队尾到队头遍历），元素不存在返回false
 
* public boolean add(E e)
同addLast

* public boolean offer(E e)
同offerLast

* public E remove()
移除队头元素，如果队列中没有元素则抛出异常

* public E poll()
移除队头元素，如果队列中没有元素则返回null

* public E element()
获取队头元素（不移除），如果队列中没有元素则抛出异常

* public E peek()
获取队头元素（不移除），如果队列中没有元素则返回null

* public void push(E e)
同addFirst，模拟栈的push操作，这里将向队头添加一个元素

* public E pop()
同removeFirst，模拟栈的pop操作，这里将移除队头元素，队列中元素为空则抛出异常

* public int size()

* public boolean isEmpty()

* public Iterator<E> iterator()

* public Iterator<E> descendingIterator()

* public boolean contains(Object o)

* public boolean remove(Object o)
元素不存在则返回false

* public void clear()

# LinkedBlockingDeque
可设置边界的阻塞队列，底层实现基于链表，不是线程安全的。
<p>Most operations run in constant time (ignoring time spent
blocking).  Exceptions include {@link #remove(Object) remove},
{@link #removeFirstOccurrence removeFirstOccurrence}, {@link
#removeLastOccurrence removeLastOccurrence}, {@link #contains
contains}, {@link #iterator iterator.remove()}, and the bulk
operations, all of which run in linear time.

该类的大多数操作都只需要常量时间O(1)，`remove`，`removeFirstOccurrence`，
`removeLastOccurrence`，`contains`，`iterator`，`iterator.remove`等方法和其它聚集操作需要线性时间O(n)

## 成员变量
```java
    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;

    /** Number of items in the deque */
    private transient int count;

    /** Maximum number of items in the deque */
    private final int capacity;

    /** Main lock guarding all access */
    final ReentrantLock lock = new ReentrantLock();

    /** Condition for waiting takes */
    private final Condition notEmpty = lock.newCondition();

    /** Condition for waiting puts */
    private final Condition notFull = lock.newCondition();

```

## 构造函数
```java
    /**
     * Creates a {@code LinkedBlockingDeque} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingDeque() {
        this(Integer.MAX_VALUE);
    }

    /**
     * Creates a {@code LinkedBlockingDeque} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this deque
     * @throws IllegalArgumentException if {@code capacity} is less than 1
     */
    public LinkedBlockingDeque(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
    }

    /**
     * Creates a {@code LinkedBlockingDeque} with a capacity of
     * {@link Integer#MAX_VALUE}, initially containing the elements of
     * the given collection, added in traversal order of the
     * collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public LinkedBlockingDeque(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock lock = this.lock;
        lock.lock(); // Never contended, but necessary for visibility
        try {
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (!linkLast(new Node<E>(e)))
                    throw new IllegalStateException("Deque full");
            }
        } finally {
            lock.unlock();
        }
    }
```

## 常用方法
* public void addFirst(E e)
添加元素到队头，如果队列已满则抛出异常，不发生阻塞

* public void addLast(E e)
添加元素到队尾部，如果队列已满则抛出异常，不发生阻塞

* public boolean offerFirst(E e)
添加元素到队头，如果队列已满则返回false，不发生阻塞

* public boolean offerLast(E e) 
添加元素到队尾，如果队列已满则返回false，不发生阻塞

* public void putFirst(E e) 
添加元素到队头，如果队列已满则抛发生阻塞直到成功放入元素

* public void putLast(E e)
添加元素到队尾，如果队列已满则抛发生阻塞直到成功放入元素

* public boolean offerFirst(E e, long timeout, TimeUnit unit)
添加元素到队头，队列已满则等待一定时间（timeout），超时后返回false

* public boolean offerLast(E e, long timeout, TimeUnit unit)
添加元素到队尾，队列已满则等待一定时间（timeout），超时后返回false

* public E removeFirst()
移除队头元素，队列为空则抛出异常，不发生阻塞

* public E removeLast()
移除队尾元素，队列为空则抛出异常，不发生阻塞

* public E pollFirst()
移除队头元素，队列为空则返回null，不发生阻塞

* public E pollLast()
移除队尾元素，队列为空则返回null，不发生阻塞

* public E takeFirst()
获取并移除队头元素，对垒为空则一直阻塞到成功取出元素

* public E takeLast()
获取并移除队尾元素，对垒为空则一直阻塞到成功取出元素

* public E pollFirst(long timeout, TimeUnit unit)

* public E pollLast(long timeout, TimeUnit unit)

* public E getFirst() 
获取（不移除）队头元素，如果队列为空则抛出异常

* public E getLast() 
获取（不移除）队尾元素，如果队列为空则抛出异常

* public E peekFirst()
同getFirst，不过队列为空时返回null，不抛出异常

* public E peekLast()

* public boolean removeFirstOccurrence(Object o) 

* public boolean removeLastOccurrence(Object o) 

* public boolean add(E e) 
同addLast

* public boolean offer(E e)
调用了offerLast

* public void put(E e) 
调用了putLast

* public boolean offer(E e, long timeout, TimeUnit unit)

* public E poll()
和removeFirst相同，但是该方法在队列为空时抛出异常

* public E take()
调用了takeFirst

* public E poll(long timeout, TimeUnit unit) 

* public E element()
调用了getFirst

* public E peek()
调用了peekFirst

* public int remainingCapacity() 

* public void push(E e)
调用了addFirst

* public E pop()
调用了removeFirst

* public boolean remove(Object o)
调用了removeFirstOccurrence

* public int size()

* public boolean contains(Object o) 

* public void clear()

* public Iterator<E> iterator()

* public Iterator<E> descendingIterator()


# ConcurrentLinkedDeque
基于链表的无边界非阻塞双向队列，是线程安全的。

## 成员变量
```java
    /**
     * A node from which the first node on list (that is, the unique node p
     * with p.prev == null && p.next != p) can be reached in O(1) time.
     * Invariants:
     * - the first node is always O(1) reachable from head via prev links
     * - all live nodes are reachable from the first node via succ()
     * - head != null
     * - (tmp = head).next != tmp || tmp != head
     * - head is never gc-unlinked (but may be unlinked)
     * Non-invariants:
     * - head.item may or may not be null
     * - head may not be reachable from the first or last node, or from tail
     */
    private transient volatile Node<E> head;

    /**
     * A node from which the last node on list (that is, the unique node p
     * with p.next == null && p.prev != p) can be reached in O(1) time.
     * Invariants:
     * - the last node is always O(1) reachable from tail via next links
     * - all live nodes are reachable from the last node via pred()
     * - tail != null
     * - tail is never gc-unlinked (but may be unlinked)
     * Non-invariants:
     * - tail.item may or may not be null
     * - tail may not be reachable from the first or last node, or from head
     */
    private transient volatile Node<E> tail;

    private static final Node<Object> PREV_TERMINATOR, NEXT_TERMINATOR;

```

## 构造方法
```java
    /**
     * Constructs an empty deque.
     */
    public ConcurrentLinkedDeque() {
        head = tail = new Node<E>(null);
    }

    /**
     * Constructs a deque initially containing the elements of
     * the given collection, added in traversal order of the
     * collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public ConcurrentLinkedDeque(Collection<? extends E> c) {
        // Copy c into a private chain of Nodes
        Node<E> h = null, t = null;
        for (E e : c) {
            checkNotNull(e);
            Node<E> newNode = new Node<E>(e);
            if (h == null)
                h = t = newNode;
            else {
                t.lazySetNext(newNode);
                newNode.lazySetPrev(t);
                t = newNode;
            }
        }
        initHeadTail(h, t);
    }
```

## 常用方法
和LinkedBlockingDeque差不多，不过方法都是非阻塞的

# Deque总结
* Deque是双向队列，队列两端都可以进行插入和移除操作
* 一般add在队列满时抛出异常，offer不抛出异常，remove和poll同理
* put和take在阻塞队列中是阻塞的
* peek和get只获取元素而不移除元素，get在队列为空时抛出异常，peek不抛出异常
* push和pop模拟栈的操作，push是往队头添加一个元素，pop是移除队头元素，队列为空或者满时都将抛出异常
