---
title: Collection Framework(1)
date: 2018-09-09 16:01:13
tags:
    - Container
    - Collection
---

{% meting "28556280" "netease" "song" %}

关于Java Collection的几个常用实现类的总结

# ArrayList
ArrayList实现了List接口，并且实现了该接口的所有可选操作（`replaceAll`，`spliterator`，`sort`）。
它的大小是可以动态变化的，相当于一个可变长度的数组，它可以存储任何类型的对象，包括null。

 The <tt>size</tt>, <tt>isEmpty</tt>, <tt>get</tt>, <tt>set</tt>,
  <tt>iterator</tt>, and <tt>listIterator</tt> operations run in constant
  time.  The <tt>add</tt> operation runs in <i>amortized constant time</i>,
  that is, adding n elements requires O(n) time.  All of the other operations
  run in linear time (roughly speaking). 

`size`, `isEmpty`, `get`, `set`, `iterator`, `listIterator`这些方法的时间复杂度都是都是O(1)，
`add(E e)`的时间复杂度是O(1)，`add(int index, E element)`的时间复杂度是O(n)，其他的方法大多数都是线性时间复杂度O(n)。

该类不是线程安全的，假设要在多线程环境下使用ArrayList，需要使用外部同步
保证对ArrayList的修改对其他线程可见，也可以使用`Collections.synchronizedList`方法使得ArrayList的实例变成线程安全的，
比如`List list = Collections.synchronizedList(new ArrayList(...))`

## fail-fast
在调用`iterator`或者`listIterator`创建了一个迭代器对象之后，假如在使用该迭代器访问元素的过程中通过除了迭代器对象的`remove`
和`add`结构化修改了（一般除了set方法外的修改都是结构化修改）ArrayList里面的元素，就会抛出`ConcurrentModificationException`异常。
在没有被正确同步的情况下，该机制有可能会失效。

## 成员变量
```angularjs
    /**
     * Default initial capacity. 默认容量为10
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
```

## 构造方法
```angularjs

    /**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

## 常用方法
* public void ensureCapacity(int minCapacity) 
增加ArrayList的容量

* public boolean isEmpty()

* public boolean contains(Object o)

* public int indexOf(Object o)

* public int lastIndexOf(Object o) 

* public E set(int index, E element)

* public E get(int index) 

* public int size() 

* public boolean add(E e)

* public void add(int index, E element)

* public E remove(int index)

* public boolean remove(Object o) 

* public void clear()

* public boolean addAll(Collection<? extends E> c) 

* public boolean addAll(int index, Collection<? extends E> c) 
从原集合的index位置开始添加c的元素，原集合的index位置后面的元素将移动到最后面

* public boolean removeAll(Collection<?> c)

* public boolean retainAll(Collection<?> c)
交集（仅保留在ArrayList对象和c里面均存在的元素）

* public Iterator<E> iterator()

* public ListIterator<E> listIterator()

# Vector
`Vector`类是线程安全的，因此多线程环境下使用`Vector`不需要外部同步。它的结构和`ArrayList`类似，
可以说它是`ArrayList`的线程安全的版本。

## 成员变量
```angularjs
    /**
     * The array buffer into which the components of the vector are
     * stored. The capacity of the vector is the length of this array buffer,
     * and is at least large enough to contain all the vector's elements.
     *
     * <p>Any array elements following the last element in the Vector are null.
     *
     * @serial
     */
    protected Object[] elementData;

    /**
     * The number of valid components in this {@code Vector} object.
     * Components {@code elementData[0]} through
     * {@code elementData[elementCount-1]} are the actual items.
     *
     * @serial
     */
    protected int elementCount;

    /**
     * The amount by which the capacity of the vector is automatically
     * incremented when its size becomes greater than its capacity.  If
     * the capacity increment is less than or equal to zero, the capacity
     * of the vector is doubled each time it needs to grow.
     *
     * @serial
     */
    protected int capacityIncrement;
```

## 构造函数
```angularjs

    /**
     * Constructs an empty vector with the specified initial capacity and
     * capacity increment.
     *
     * @param   initialCapacity     the initial capacity of the vector
     * @param   capacityIncrement   the amount by which the capacity is
     *                              increased when the vector overflows
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }

    /**
     * Constructs an empty vector with the specified initial capacity and
     * with its capacity increment equal to zero.
     *
     * @param   initialCapacity   the initial capacity of the vector
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public Vector(int initialCapacity) {
        this(initialCapacity, 0);
    }

    /**
     * Constructs an empty vector so that its internal data array
     * has size {@code 10} and its standard capacity increment is
     * zero.
     */
    public Vector() {
        this(10);
    }

    /**
     * Constructs a vector containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this
     *       vector
     * @throws NullPointerException if the specified collection is null
     * @since   1.2
     */
    public Vector(Collection<? extends E> c) {
        elementData = c.toArray();
        elementCount = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
    }
```

## 常用方法
* public synchronized void setSize(int newSize)
设置链表的elementCount，如果要设置的elementCount比当前的elementCount要大，则往里面填充null对象，否则将清空要设置的大小以外的元素

* public synchronized int capacity()
获取当前的容量（elementData.length）

* public synchronized int size()
获取当前存储的元素个数（elementCount）

* public synchronized boolean isEmpty()

* public Enumeration<E> elements()
返回当前存储元素对象的枚举，和Iterator功能差不多，用于遍历存储的元素。

* public boolean contains(Object o)

* public int indexOf(Object o)
获取特定对象在链表中首次出现的索引，若元素不在链表中则返回-1

* public synchronized int indexOf(Object o, int index)
从指定位置（index）开始获取特定对象在链表中首次出现的索引，不存在返回-1

* public synchronized int lastIndexOf(Object o)

* public synchronized int lastIndexOf(Object o, int index)

* public synchronized E elementAt(int index)

* public synchronized E firstElement() 

* public synchronized E lastElement() 

* public synchronized void setElementAt(E obj, int index)

* public synchronized void removeElementAt(int index) 

* public synchronized void insertElementAt(E obj, int index)

* public synchronized void addElement(E obj)
添加一个元素到尾部

* public synchronized boolean removeElement(Object obj) 

* public synchronized void removeAllElements()

* public synchronized E get(int index) 

* public synchronized E set(int index, E element)

* public synchronized boolean add(E e) 

* public boolean remove(Object o)

* public void add(int index, E element)

* public synchronized E remove(int index)

* public void clear()

* public synchronized boolean containsAll(Collection<?> c)
如果链表包含c中的所以元素，则返回true

* public synchronized boolean addAll(Collection<? extends E> c)

* public synchronized boolean addAll(int index, Collection<? extends E> c) 

* public synchronized boolean removeAll(Collection<?> c)

* public synchronized boolean retainAll(Collection<?> c)
取交集（在该链表和c中均包含的元素）


# LinkedList

LinkedList实现了List和Deque接口，它允许存放任何类型的对象，包括null，该类不是线程安全的。
使用索引去访问存储的元素时，将从第一个节点开始遍历到相对应的节点，而不是直接通过索引就能取出元素，因为底层结构改变了，
变成了一个个通过next和prev变量相连接的Node，Node的结构如下：
```angularjs
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

## 成员变量
```angularjs
    transient int size = 0;  //存储的元素个数

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;  //第一个节点

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;  //最后一个节点
```

## 构造方法
```angularjs
    /**
     * Constructs an empty list.
     */
    public LinkedList() {
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param  c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

## 常用方法

* public E getFirst()

* public E getLast()

* public E removeFirst()

* public E removeLast()

* public void addFirst(E e)

* public void addLast(E e)

* public boolean contains(Object o)

* public int size()

* public boolean add(E e)
添加一个元素到链表的尾部

* public boolean remove(Object o)
移除特定元素（假设有多个相同的元素，将移除在链表最前面出现的元素）

* public boolean addAll(Collection<? extends E> c)

* public boolean addAll(int index, Collection<? extends E> c)

* public void clear()

* public E get(int index)

* public E set(int index, E element)

* public void add(int index, E element)

* public E remove(int index) 

* public int indexOf(Object o)

* public int lastIndexOf(Object o)

* public E peek()
取最后面的元素，只是获取该元素，不移除，当元素不存在时，返回null

* public E peekFirst()

* public E peekLast() 

* public E element()
和peek方法一样，不过当元素不存在时候，抛出NoSuchElementException

* public E poll()
获取并移除最后面的元素，当元素不存在时，返回null

* public E pollFirst() 

* public E pollLast()

* public E remove()
和poll方法一样，不过当元素不存在时候，抛出NoSuchElementException

* public boolean offer(E e)
添加元素到最后面

* public boolean offerFirst(E e)
添加元素到最前面

* public boolean offerLast(E e)
添加元素到最后面 

* public void push(E e)
添加元素到最前面

* public E pop()
弹出最前面的元素，push和pop是栈的方法，因为前面的push是添加元素到最前面的，所以现在pop也应该弹出
最前面的元素才符合FIFO

* public boolean removeFirstOccurrence(Object o)
移除第一次出现的特定元素，从fistNode开始到lastNode结束

* public boolean removeLastOccurrence(Object o)

# CopyOnWriteArrayList
和ArrayList类似，不过它是线程安全的，它对的`add`和`set`等操作都将重新复制原来的数组元素并创建新的数组，
因此开销非常大，一般只用来遍历和查找元素，如果要频繁地增加或改动元素则不应该使用该类。它允许插入null元素。

## 成员变量
```angularjs
    /** The lock protecting all mutators */
    final transient ReentrantLock lock = new ReentrantLock();

    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```

## 构造方法
```angularjs
    /**
     * Creates an empty list.
     */
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }

    /**
     * Creates a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection of initially held elements
     * @throws NullPointerException if the specified collection is null
     */
    public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            elements = c.toArray();
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elements.getClass() != Object[].class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }

    /**
     * Creates a list holding a copy of the given array.
     *
     * @param toCopyIn the array (a copy of this array is used as the
     *        internal array)
     * @throws NullPointerException if the specified array is null
     */
    public CopyOnWriteArrayList(E[] toCopyIn) {
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }
```

## 常用方法
参考ArrayList

# List总结
* `ArrayList`不是线程安全的，`Vector`是线程安全的
* `ArayList`的查找操作只需要常数时间，`LinkedList`则需要线性时间
* `LinkedList`同时实现了`List`和`Deque`接口
* `List`中可以包含重复的元素
* `CopyOnWriteArrayList`一般用于多线程中频繁地访问元素的情景

# HashMap
`HashMap`的实现已经在[前几篇博客](https://plentymore.github.io/2018/08/14/HashMap/)介绍过了。
`HashMap`不是线程安全的，它可以存放null键值对

## 构造函数
```angularjs

    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity.
     * @throws IllegalArgumentException if the initial capacity is negative.
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    /**
     * Constructs a new <tt>HashMap</tt> with the same mappings as the
     * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
     * default load factor (0.75) and an initial capacity sufficient to
     * hold the mappings in the specified <tt>Map</tt>.
     *
     * @param   m the map whose mappings are to be placed in this map
     * @throws  NullPointerException if the specified map is null
     */
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

## 常用方法
* public int size()

* public boolean isEmpty()

* public V get(Object key)

* public boolean containsKey(Object key)

* public V put(K key, V value)

* public void putAll(Map<? extends K, ? extends V> m)

* public V remove(Object key) 

* public void clear()

* public boolean containsValue(Object value)

* public Set<K> keySet()
获取存储的所有键值对的键的Set对象，因为键是没有重复的，因此返回一个Set对象，而不是List

* public Collection<V> values()
获取存储的所有键值对的值的Collection对象，因为值可以是可以重复的任意对象，因此返回一个Collection对象

* public Set<Map.Entry<K,V>> entrySet()
获取存储的所有键值对，键值对被封装进一个Set对象后返回

* public V getOrDefault(Object key, V defaultValue) 
获取某个key的值，如果key不存在则返回默认值（defaultValue）

* public V putIfAbsent(K key, V value)
当要存放的键值对的键不存在于Map中时才将键值对存放进Map中

* public boolean remove(Object key, Object value)
当指定的值和要移除的键值对的值相匹配时才移除该键值对

* public boolean replace(K key, V oldValue, V newValue)

* public V replace(K key, V value) 

# LinkedHashMap
`LinkedHashMap`继承了`HashMap`并且实现了`Map`接口，它的迭代顺序是可预测的（通过它的iterator方法得到的迭代器迭代对象的时候是按照元素插入的时间顺序进行迭代的）。
它一般用来复制Map对象，并能保证复制后的对象的元素顺序和复制前的对象一致。

## 构造方法
```angularjs
    /**
     * Constructs an empty insertion-ordered <tt>LinkedHashMap</tt> instance
     * with the specified initial capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }

    /**
     * Constructs an empty insertion-ordered <tt>LinkedHashMap</tt> instance
     * with the specified initial capacity and a default load factor (0.75).
     *
     * @param  initialCapacity the initial capacity
     * @throws IllegalArgumentException if the initial capacity is negative
     */
    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }

    /**
     * Constructs an empty insertion-ordered <tt>LinkedHashMap</tt> instance
     * with the default initial capacity (16) and load factor (0.75).
     */
    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

    /**
     * Constructs an insertion-ordered <tt>LinkedHashMap</tt> instance with
     * the same mappings as the specified map.  The <tt>LinkedHashMap</tt>
     * instance is created with a default load factor (0.75) and an initial
     * capacity sufficient to hold the mappings in the specified map.
     *
     * @param  m the map whose mappings are to be placed in this map
     * @throws NullPointerException if the specified map is null
     */
    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super();
        accessOrder = false;
        putMapEntries(m, false);
    }

    /**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

## 常用方法
和HashMap一致

# Hashtable
`Hashtable`功能和HashMap差不多，但Hashtable是线程安全的。Hashtable的实现也在[这里](https://plentymore.github.io/2018/08/15/Hashtable/)介绍过了。
它不能存放null键值对。

## 构造方法
```angularjs

    /**
     * Constructs a new, empty hashtable with the specified initial
     * capacity and the specified load factor.
     *
     * @param      initialCapacity   the initial capacity of the hashtable.
     * @param      loadFactor        the load factor of the hashtable.
     * @exception  IllegalArgumentException  if the initial capacity is less
     *             than zero, or if the load factor is nonpositive.
     */
    public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }

    /**
     * Constructs a new, empty hashtable with the specified initial capacity
     * and default load factor (0.75).
     *
     * @param     initialCapacity   the initial capacity of the hashtable.
     * @exception IllegalArgumentException if the initial capacity is less
     *              than zero.
     */
    public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }

    /**
     * Constructs a new, empty hashtable with a default initial capacity (11)
     * and load factor (0.75).
     */
    public Hashtable() {
        this(11, 0.75f);
    }

    /**
     * Constructs a new hashtable with the same mappings as the given
     * Map.  The hashtable is created with an initial capacity sufficient to
     * hold the mappings in the given Map and a default load factor (0.75).
     *
     * @param t the map whose mappings are to be placed in this map.
     * @throws NullPointerException if the specified map is null.
     * @since   1.2
     */
    public Hashtable(Map<? extends K, ? extends V> t) {
        this(Math.max(2*t.size(), 11), 0.75f);
        putAll(t);
    }
```

## 常用方法
* public synchronized int size()

* public synchronized boolean isEmpty()

* public synchronized Enumeration<K> keys()
获取存储的所有键值对的键，封装成Enumeration类型后返回

* public synchronized Enumeration<V> elements()
获取存储的所有键值对的值，封装成Enumeration类型后返回

* public synchronized boolean contains(Object value)

* public boolean containsValue(Object value)

* public synchronized boolean containsKey(Object key)

* public synchronized V get(Object key)

* public synchronized V put(K key, V value)

* public synchronized V remove(Object key)

* public synchronized boolean remove(Object key, Object value)
当key和value都与Map里面对象的元素匹配时才移除元素

* public synchronized void putAll(Map<? extends K, ? extends V> t)

* public synchronized void clear()

* public Set<K> keySet() 

* public Set<Map.Entry<K,V>> entrySet()

* public Collection<V> values()

* public synchronized V getOrDefault(Object key, V defaultValue) 

* public synchronized V putIfAbsent(K key, V value)

* public synchronized boolean replace(K key, V oldValue, V newValue) 

* public synchronized V replace(K key, V value) 

# TreeMap
`TreeMap`是以红黑色为底层结构的Map借口实现类，功能和其他的Map实现了基本一致。该类不是线程安全的，
要在多线程中使用它需要使用外部同步。它不能存放为null的键，但可以存放为null的值

This implementation provides guaranteed log(n) time cost for the
 {@code containsKey}, {@code get}, {@code put} and {@code remove}
 operations
 
 `containsKey`， `get`，`put`，`remove`等方法的时间复杂度是O(log(n))。该类存储的元素是有序的，
 一般通过自然顺序进行排序，如果在构造函数中提供了自定义的Comparator，则按照自定义的Comparator进行排序。
 
 ## 成员变量
 ```angularjs
    /**
     * The comparator used to maintain order in this tree map, or
     * null if it uses the natural ordering of its keys.
     *
     * @serial
     */
    private final Comparator<? super K> comparator;

    private transient Entry<K,V> root;

    /**
     * The number of entries in the tree
     */
    private transient int size = 0;

    /**
     * The number of structural modifications to the tree.
     */
    private transient int modCount = 0;
```

## 构造函数
```angularjs
    /**
     * Constructs a new, empty tree map, using the natural ordering of its
     * keys.  All keys inserted into the map must implement the {@link
     * Comparable} interface.  Furthermore, all such keys must be
     * <em>mutually comparable</em>: {@code k1.compareTo(k2)} must not throw
     * a {@code ClassCastException} for any keys {@code k1} and
     * {@code k2} in the map.  If the user attempts to put a key into the
     * map that violates this constraint (for example, the user attempts to
     * put a string key into a map whose keys are integers), the
     * {@code put(Object key, Object value)} call will throw a
     * {@code ClassCastException}.
     */
    public TreeMap() {
        comparator = null;
    }

    /**
     * Constructs a new, empty tree map, ordered according to the given
     * comparator.  All keys inserted into the map must be <em>mutually
     * comparable</em> by the given comparator: {@code comparator.compare(k1,
     * k2)} must not throw a {@code ClassCastException} for any keys
     * {@code k1} and {@code k2} in the map.  If the user attempts to put
     * a key into the map that violates this constraint, the {@code put(Object
     * key, Object value)} call will throw a
     * {@code ClassCastException}.
     *
     * @param comparator the comparator that will be used to order this map.
     *        If {@code null}, the {@linkplain Comparable natural
     *        ordering} of the keys will be used.
     */
    public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }

    /**
     * Constructs a new tree map containing the same mappings as the given
     * map, ordered according to the <em>natural ordering</em> of its keys.
     * All keys inserted into the new map must implement the {@link
     * Comparable} interface.  Furthermore, all such keys must be
     * <em>mutually comparable</em>: {@code k1.compareTo(k2)} must not throw
     * a {@code ClassCastException} for any keys {@code k1} and
     * {@code k2} in the map.  This method runs in n*log(n) time.
     *
     * @param  m the map whose mappings are to be placed in this map
     * @throws ClassCastException if the keys in m are not {@link Comparable},
     *         or are not mutually comparable
     * @throws NullPointerException if the specified map is null
     */
    public TreeMap(Map<? extends K, ? extends V> m) {
        comparator = null;
        putAll(m);
    }

    /**
     * Constructs a new tree map containing the same mappings and
     * using the same ordering as the specified sorted map.  This
     * method runs in linear time.
     *
     * @param  m the sorted map whose mappings are to be placed in this map,
     *         and whose comparator is to be used to sort this map
     * @throws NullPointerException if the specified map is null
     */
    public TreeMap(SortedMap<K, ? extends V> m) {
        comparator = m.comparator();
        try {
            buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
    }
```

## 常用方法
* public int size()

* public boolean containsKey(Object key)

* public boolean containsValue(Object value) 

* public V get(Object key) 

* public Comparator<? super K> comparator()
获取自定义的Comparator

* public K firstKey()

* public K lastKey()

* public void putAll(Map<? extends K, ? extends V> map)

* public V put(K key, V value) 

* public V remove(Object key)

* public void clear()

* public Map.Entry<K,V> firstEntry()
获取第一个键值对的Entry对象

* public Map.Entry<K,V> lastEntry()
获取最后一个键值对的Entry对象

* public Map.Entry<K,V> pollFirstEntry()
移除并返回Map中最前面的一个元素

* public Map.Entry<K,V> pollLastEntry()
移除并返回Map中最后面的一个元素

* public Map.Entry<K,V> lowerEntry(K key)
返回key小于指定key的键值对，返回的键值对的键同时也是Map中的最大的键

* public K lowerKey(K key)
返回key小于指定key的键，返回的键同时也是Map中的最大的键

* public Map.Entry<K,V> floorEntry(K key)
返回key小于或等于指定key的键值对，返回的键值对的键同时也是Map中的最大的键

* public K floorKey(K key)
返回key小于等于指定key的键，返回的键同时也是Map中的最大的键

* public Map.Entry<K,V> ceilingEntry(K key)
返回key大于或等于指定key的键值对，返回的键值对的键同时也是Map中的最小的键

* public K ceilingKey(K key)
返回key大于等于指定key的键，返回的键同时也是Map中的最小的键

* public Map.Entry<K,V> higherEntry(K key)
返回key大于指定key的键值对，返回的键值对的键同时也是Map中的最小的键

* public K higherKey(K key)
返回key大于指定key的键，返回的键同时也是Map中的最小的键

* public Set<K> keySet()

* public NavigableSet<K> navigableKeySet()
返回Map中的所有key，key按照原本的顺序排列，封装成NavigableSet类型后返回

* public NavigableSet<K> descendingKeySet()
返回Map中的所有key，key按照降序排列，封装成NavigableSet类型后返回

* public Collection<V> values() 
返回Map中的所有值，值按照升序排列，封装成Collection类型后返回

* public Set<Map.Entry<K,V>> entrySet()
返回Map中的所有键值对，按照升序排列，封装成Set类型后返回

* public NavigableMap<K, V> descendingMap()
返回Map中所有键值对，按照倒叙排列，封装成NavigableMap类型后返回

* public boolean replace(K key, V oldValue, V newValue)
当指定的key和oldValue和Map中的key-value都对应时替换oldValue为newValue

* public V replace(K key, V value)
替换指定key的value

# ConcurrentHashMap
这是线程安全版的`HashMap`，主要在并发环境下使用，它是线程安全的。它的实现比`HashMap`要复杂很多，
由于使用了CAS来实现同步，在并发环境它的性能要比Hashtable高很多，因此在并发环境下建议使用`ConcurrentHashMap`，
它不允许存放null键值对。

## 成员变量
```angularjs
    /* ---------------- Constants -------------- */

    /**
     * The largest possible table capacity.  This value must be
     * exactly 1<<30 to stay within Java array allocation and indexing
     * bounds for power of two table sizes, and is further required
     * because the top two bits of 32bit hash fields are used for
     * control purposes.
     */
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The default initial table capacity.  Must be a power of 2
     * (i.e., at least 1) and at most MAXIMUM_CAPACITY.
     */
    private static final int DEFAULT_CAPACITY = 16;

    /**
     * The largest possible (non-power of two) array size.
     * Needed by toArray and related methods.
     */
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * The default concurrency level for this table. Unused but
     * defined for compatibility with previous versions of this class.
     */
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    /**
     * The load factor for this table. Overrides of this value in
     * constructors affect only the initial table capacity.  The
     * actual floating point value isn't normally used -- it is
     * simpler to use expressions such as {@code n - (n >>> 2)} for
     * the associated resizing threshold.
     */
    private static final float LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2, and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * The value should be at least 4 * TREEIFY_THRESHOLD to avoid
     * conflicts between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * Minimum number of rebinnings per transfer step. Ranges are
     * subdivided to allow multiple resizer threads.  This value
     * serves as a lower bound to avoid resizers encountering
     * excessive memory contention.  The value should be at least
     * DEFAULT_CAPACITY.
     */
    private static final int MIN_TRANSFER_STRIDE = 16;

    /**
     * The number of bits used for generation stamp in sizeCtl.
     * Must be at least 6 for 32bit arrays.
     */
    private static int RESIZE_STAMP_BITS = 16;

    /**
     * The maximum number of threads that can help resize.
     * Must fit in 32 - RESIZE_STAMP_BITS bits.
     */
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    /**
     * The bit shift for recording size stamp in sizeCtl.
     */
    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    /*
     * Encodings for Node hash fields. See above for explanation.
     */
    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

    /** Number of CPUS, to place bounds on some sizings */
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /** For serialization compatibility. */
    private static final ObjectStreamField[] serialPersistentFields = {
        new ObjectStreamField("segments", Segment[].class),
        new ObjectStreamField("segmentMask", Integer.TYPE),
        new ObjectStreamField("segmentShift", Integer.TYPE)
    };
    
        /* ---------------- Fields -------------- */
    
        /**
         * The array of bins. Lazily initialized upon first insertion.
         * Size is always a power of two. Accessed directly by iterators.
         */
        transient volatile Node<K,V>[] table;
    
        /**
         * The next table to use; non-null only while resizing.
         */
        private transient volatile Node<K,V>[] nextTable;
    
        /**
         * Base counter value, used mainly when there is no contention,
         * but also as a fallback during table initialization
         * races. Updated via CAS.
         */
        private transient volatile long baseCount;
    
        /**
         * Table initialization and resizing control.  When negative, the
         * table is being initialized or resized: -1 for initialization,
         * else -(1 + the number of active resizing threads).  Otherwise,
         * when table is null, holds the initial table size to use upon
         * creation, or 0 for default. After initialization, holds the
         * next element count value upon which to resize the table.
         */
        private transient volatile int sizeCtl;
    
        /**
         * The next table index (plus one) to split while resizing.
         */
        private transient volatile int transferIndex;
    
        /**
         * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
         */
        private transient volatile int cellsBusy;
    
        /**
         * Table of counter cells. When non-null, size is a power of 2.
         */
        private transient volatile CounterCell[] counterCells;
    
        // views
        private transient KeySetView<K,V> keySet;
        private transient ValuesView<K,V> values;
        private transient EntrySetView<K,V> entrySet;
```

## 构造函数
```angularjs

    /**
     * Creates a new, empty map with the default initial table size (16).
     */
    public ConcurrentHashMap() {
    }

    /**
     * Creates a new, empty map with an initial table size
     * accommodating the specified number of elements without the need
     * to dynamically resize.
     *
     * @param initialCapacity The implementation performs internal
     * sizing to accommodate this many elements.
     * @throws IllegalArgumentException if the initial capacity of
     * elements is negative
     */
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }

    /**
     * Creates a new map with the same mappings as the given map.
     *
     * @param m the map
     */
    public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
    }

    /**
     * Creates a new, empty map with an initial table size based on
     * the given number of elements ({@code initialCapacity}) and
     * initial table density ({@code loadFactor}).
     *
     * @param initialCapacity the initial capacity. The implementation
     * performs internal sizing to accommodate this many elements,
     * given the specified load factor.
     * @param loadFactor the load factor (table density) for
     * establishing the initial table size
     * @throws IllegalArgumentException if the initial capacity of
     * elements is negative or the load factor is nonpositive
     *
     * @since 1.6
     */
    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

    /**
     * Creates a new, empty map with an initial table size based on
     * the given number of elements ({@code initialCapacity}), table
     * density ({@code loadFactor}), and number of concurrently
     * updating threads ({@code concurrencyLevel}).
     *
     * @param initialCapacity the initial capacity. The implementation
     * performs internal sizing to accommodate this many elements,
     * given the specified load factor.
     * @param loadFactor the load factor (table density) for
     * establishing the initial table size
     * @param concurrencyLevel the estimated number of concurrently
     * updating threads. The implementation may use this value as
     * a sizing hint.
     * @throws IllegalArgumentException if the initial capacity is
     * negative or the load factor or concurrencyLevel are
     * nonpositive
     */
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

## 常用方法
常用方法和Hashtable一样

# IdentityHashMap
`IdentityHashMap`是一种特殊的Map，它的特别之处在于，在判断两个key是否相等的时候，
只有当key1 == key2的时候（使用`==`操作符）才认为这两个key是相等的，因此它的判断方式是判断引用相等，而不是对象相等。
该类的详细用途可以查看[文档](https://docs.oracle.com/javase/8/docs/api/)

# Map总结
* `ConcurrentHashMap`和`Hashtable`是线程安全的，其它的不是
* `ConcurrentHashMap`和`Hashtable`不允许存放null键值对，`HashMap`可以，`TreeMap`可以不可存放null键，可以存放null值
* `TreeMap`是有序（按照自然顺序或者用户自定义排序规则排序）的，其他的一般是无序的（如返回的`Iterator`不保证迭代元素顺序一致）
* `HashMap`和`ConcurrentHashMap`在JDK1.8开始，在Hash冲突严重的时候会将链表结构转换成红黑树结构，因此它们的查找操作的最坏时间复杂度为O(log(n))

# HashSet
`HashSet`是`Set`接口的一种Hash实现，和`List`接口不同，`Set`接口不包含重复的元素。
`HashSet`不保证迭代顺序，它不是线程安全的，允许添加null元素

## 成员变量
```angularjs
    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
```

## 构造方法
```angularjs
    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * default initial capacity (16) and load factor (0.75).
     */
    public HashSet() {
        map = new HashMap<>();
    }

    /**
     * Constructs a new set containing the elements in the specified
     * collection.  The <tt>HashMap</tt> is created with default load factor
     * (0.75) and an initial capacity sufficient to contain the elements in
     * the specified collection.
     *
     * @param c the collection whose elements are to be placed into this set
     * @throws NullPointerException if the specified collection is null
     */
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * the specified initial capacity and the specified load factor.
     *
     * @param      initialCapacity   the initial capacity of the hash map
     * @param      loadFactor        the load factor of the hash map
     * @throws     IllegalArgumentException if the initial capacity is less
     *             than zero, or if the load factor is nonpositive
     */
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    /**
     * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
     * the specified initial capacity and default load factor (0.75).
     *
     * @param      initialCapacity   the initial capacity of the hash table
     * @throws     IllegalArgumentException if the initial capacity is less
     *             than zero
     */
    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    /**
     * Constructs a new, empty linked hash set.  (This package private
     * constructor is only used by LinkedHashSet.) The backing
     * HashMap instance is a LinkedHashMap with the specified initial
     * capacity and the specified load factor.
     *
     * @param      initialCapacity   the initial capacity of the hash map
     * @param      loadFactor        the load factor of the hash map
     * @param      dummy             ignored (distinguishes this
     *             constructor from other int, float constructor.)
     * @throws     IllegalArgumentException if the initial capacity is less
     *             than zero, or if the load factor is nonpositive
     */
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

## 常用方法
* public int size()

* public boolean isEmpty()

* public boolean contains(Object o)

* public boolean add(E e)
添加一个元素到Set，如果该元素已经存在，则不添加

* public boolean remove(Object o)

* public void clear()

# TreeSet
`NavigableSet`的一种实现，它的实现基于`TreeMap`，其实就是没有值的`TreeMap`，
它的元素是有序（按照自然顺序或者用户自定义的排序规则排序）的。
  <p>This implementation provides guaranteed log(n) time cost for the basic
  operations ({@code add}, {@code remove} and {@code contains}).

`add`，`remove`，`contains`等基本操作的时间复杂度均为log(n)，它不是线程安全的。

## 成员变量
```angularjs
    /**
     * The backing map.
     */
    private transient NavigableMap<E,Object> m;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

```

## 构造函数
```angularjs
    /**
     * Constructs a set backed by the specified navigable map.
     */
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    /**
     * Constructs a new, empty tree set, sorted according to the
     * natural ordering of its elements.  All elements inserted into
     * the set must implement the {@link Comparable} interface.
     * Furthermore, all such elements must be <i>mutually
     * comparable</i>: {@code e1.compareTo(e2)} must not throw a
     * {@code ClassCastException} for any elements {@code e1} and
     * {@code e2} in the set.  If the user attempts to add an element
     * to the set that violates this constraint (for example, the user
     * attempts to add a string element to a set whose elements are
     * integers), the {@code add} call will throw a
     * {@code ClassCastException}.
     */
    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    /**
     * Constructs a new, empty tree set, sorted according to the specified
     * comparator.  All elements inserted into the set must be <i>mutually
     * comparable</i> by the specified comparator: {@code comparator.compare(e1,
     * e2)} must not throw a {@code ClassCastException} for any elements
     * {@code e1} and {@code e2} in the set.  If the user attempts to add
     * an element to the set that violates this constraint, the
     * {@code add} call will throw a {@code ClassCastException}.
     *
     * @param comparator the comparator that will be used to order this set.
     *        If {@code null}, the {@linkplain Comparable natural
     *        ordering} of the elements will be used.
     */
    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    /**
     * Constructs a new tree set containing the elements in the specified
     * collection, sorted according to the <i>natural ordering</i> of its
     * elements.  All elements inserted into the set must implement the
     * {@link Comparable} interface.  Furthermore, all such elements must be
     * <i>mutually comparable</i>: {@code e1.compareTo(e2)} must not throw a
     * {@code ClassCastException} for any elements {@code e1} and
     * {@code e2} in the set.
     *
     * @param c collection whose elements will comprise the new set
     * @throws ClassCastException if the elements in {@code c} are
     *         not {@link Comparable}, or are not mutually comparable
     * @throws NullPointerException if the specified collection is null
     */
    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    /**
     * Constructs a new tree set containing the same elements and
     * using the same ordering as the specified sorted set.
     *
     * @param s sorted set whose elements will comprise the new set
     * @throws NullPointerException if the specified sorted set is null
     */
    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
```

## 常用方法
* public Iterator<E> iterator()

* public Iterator<E> descendingIterator()

* public NavigableSet<E> descendingSet()

* public int size()

* public boolean isEmpty()

* public boolean contains(Object o)

* public boolean add(E e)

* public boolean remove(Object o)

* public void clear()

# LinkedHashSet
`LinkedHashSet`的迭代是有序的（通过它的iterator方法得到的迭代器迭代元素的顺序和插入元素的时间顺序一致），
它继承了`HashMap`类并实现了`Set`接口，一般用来复制一个Set对象，并且要保持复制后的元素顺序和原来的Set对象一致，比如：
```angularjs
void foo(Set s) {
    Set copy = new LinkedHashSet(s);
    ...
}
```

## 成员变量
没有自己的成员变量

## 构造器
```angularjs

    /**
     * Constructs a new, empty linked hash set with the specified initial
     * capacity and load factor.
     *
     * @param      initialCapacity the initial capacity of the linked hash set
     * @param      loadFactor      the load factor of the linked hash set
     * @throws     IllegalArgumentException  if the initial capacity is less
     *               than zero, or if the load factor is nonpositive
     */
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    /**
     * Constructs a new, empty linked hash set with the specified initial
     * capacity and the default load factor (0.75).
     *
     * @param   initialCapacity   the initial capacity of the LinkedHashSet
     * @throws  IllegalArgumentException if the initial capacity is less
     *              than zero
     */
    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    /**
     * Constructs a new, empty linked hash set with the default initial
     * capacity (16) and load factor (0.75).
     */
    public LinkedHashSet() {
        super(16, .75f, true);
    }

    /**
     * Constructs a new linked hash set with the same elements as the
     * specified collection.  The linked hash set is created with an initial
     * capacity sufficient to hold the elements in the specified collection
     * and the default load factor (0.75).
     *
     * @param c  the collection whose elements are to be placed into
     *           this set
     * @throws NullPointerException if the specified collection is null
     */
    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }
``` 

## 常用方法
继承了HashSet的方法


# Set总结
* `Set`不包含重复元素
* `HashSet`是无序的(迭代元素的时候不会按照插入元素的顺序进行迭代)，`TreeSet`和`LinkedHashSet`是有序的

