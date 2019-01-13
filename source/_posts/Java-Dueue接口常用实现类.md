---
title: Java Dueue接口常用实现类
date: 2019-01-09 18:56:59
tags:
    - Collection
---

[Deque](https://docs.oracle.com/javase/8/docs/api/java/util/Deque.html)表示双向队列，可以从队首或者队尾插入元素。

## 接口

```java
public interface Deque<E> extends Queue<E> {
    // 添加元素到队列头部，如果添加失败，则抛出异常
	void addFirst(E e);

    // 添加元素到队尾，如果添加失败，则抛出异常
	void addLast(E e);

    // 和addFirst一样，不同之处是该方法有返回值，由于容量不足添加失败会返回false
	boolean offerFirst(E e);

    // 和addLast一样，不同之处是该方法有返回值，由于容量不足添加失败会返回false
	boolean offerLast(E e);

    // 移除并返回队首元素，如果队列为空则抛出异常，不返回null
    E removeFirst();

    // 移除并返回队尾元素，如果队列为空则抛出异常，不返回null
    E removeLast();

    // 和removeFirst方法一样，不过该方法在队列为空的时候返回null，不抛出异常
    E pollFirst();

    // 和removeLast方法一样，不过该方法在队列为空的时候返回null，不抛出异常
    E pollLast();

    // 返回队首元素，如果队列为空则抛出异常，不返回null
    E getFirst();
    
    // 返回队尾元素，如果队列为空则抛出异常，不返回null
    E getLast();

    // 和getFirst方法一样，不过该方法在队列为空的时候返回null，不抛出异常
    E peekFirst();

    // 和getLast方法一样，不过该方法在队列为空的时候返回null，不抛出异常
    E peekLast();

    // 移除在队列中首次出现的特定元素（从队头到队尾），如果元素不存在，则返回false
    boolean removeFirstOccurrence(Object o);

    // 移除在队列中最后出现的特定元素（从队头到队尾），如果元素不存在，则返回false
    boolean removeLastOccurrence(Object o);

    // *** Queue methods ***

    // 添加元素到队列尾部，如果队列容量不足，则抛出异常，不返回false
    boolean add(E e);

    // 和add方法一样，不过该方法在队列容量不足的时候返回false，不抛出异常
    boolean offer(E e);
    
    // 移除队头元素，这些方法和Queue里面的含义是一样的
    E remove();

    E poll();

    E element();

    E peek();

    // *** Stack methods ***
    // 双向队列可以当作一个栈来使用

    // 把一个元素入栈，实际上就是添加到双向队列的首部
    // 这个方法等同于addFirst
    void push(E e);

    // 弹出栈顶元素，实际上就是双向队列中的队尾元素
    // 该方法等同于removeFirst
    E pop();

    // 这是Collection里面的remove方法，不过在这里该方法有不同的含义
    // 这个方法将移除双向队列中首次出现的元素（从队首到队尾）
    // 等同于removeFirstOccurrence
    boolean remove(Object o);
}
```

Type of Operation | First Element (Beginning of the Deque instance) | Last Element (End of the Deque instance)
-------------     | ---------- | ----------- | -----------
Insert | addFirst(e)<br>offerFirst(e) | addLast(e)<br>offerLast(e)
Remove | removeFirst()<br>pollFirst() | removeLast()<br>pollLast()
Examine | getFirst()<br>peekFirst() | getLast()<br>peekLast()

## 非线程安全的实现类

### ArrayDeque

[ArrayDeque](https://docs.oracle.com/javase/8/docs/api/java/util/ArrayDeque.html)是基于数组实现的双向队列，它是有边界的。不允许存放`null`元素

* 构造器
```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
{
    public ArrayDeque() {
        elements = new Object[16];
    }

    public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }

    public ArrayDeque(Collection<? extends E> c) {
        allocateElements(c.size());
        addAll(c);
    }	
}
```

* 默认值
最小的初始容量为8，即在构造器指定初始容量的时候，如果小于8，初始容量将会是8。使用无参构造器的情况下，初始容量默认为16。
```java
    private static final int MIN_INITIAL_CAPACITY = 8;
```

* 底层存储结构
底层存储结构为数组，扩容的时候，新容量变成旧容量\*2（`int newCapacity = n << 1;`）
```java
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
```

### LinkedList

[LinkedList](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedList.html)实现了`List`接口，同时也实现了`Deque`接口。允许存放`null`元素

* 构造器
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    public LinkedList() {
    }

    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }	
}	
```

* 默认值
无

* 底层存储结果
底层为链表，所以没有容量限制
```java

    transient Node<E> first;


    transient Node<E> last;

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
    }
```

## 线程安全的实现类

### ConcurrentLinkedDeque
[ConcurrentLinkedDeque](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentLinkedDeque.html)是无边界的双向队列，不允许插入`null`元素。它使用了无锁算法，并发效率高。

* 构造器
```java
public class ConcurrentLinkedDeque<E>
    extends AbstractCollection<E>
    implements Deque<E>, java.io.Serializable {

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
}
```

* 默认值
无

* 底层存储结构
底层存储结构为链表
```java
    private transient volatile Node<E> head;

    private transient volatile Node<E> tail;

    static final class Node<E> {
        volatile Node<E> prev;
        volatile E item;
        volatile Node<E> next;
    }
```

### LinkedBlockingDeque

[LinkedBlockingDeque](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedBlockingDeque.html)是阻塞双向队列，它既可以是有边界的，也可以是无边界的，和`LinkedBlockingQueue`类似。它使用了AQS框架实现的锁来保证线程安全。不允许存放`null`

* 构造器
```java
public class LinkedBlockingDeque<E>
    extends AbstractQueue<E>
    implements BlockingDeque<E>, java.io.Serializable {

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
}
```

* 默认值
使用无参构造器的时候默认的容量为Integer.MAX_VALUE，即默认为无边界的
```java
    /** Main lock guarding all access */
    final ReentrantLock lock = new ReentrantLock();

    /** Condition for waiting takes */
    private final Condition notEmpty = lock.newCondition();

    /** Condition for waiting puts */
    private final Condition notFull = lock.newCondition();
```

* 底层存储结构
底层存储结构为链表。
```java
    /** Doubly-linked list node class */
    static final class Node<E> {
        E item;
        Node<E> prev;
        Node<E> next;
    }
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
```
