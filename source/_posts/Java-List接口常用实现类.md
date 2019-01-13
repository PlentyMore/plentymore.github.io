---
title: Java List接口常用实现类
date: 2019-01-09 18:56:12
tags:
    - Collection
---

[List](https://docs.oracle.com/javase/8/docs/api/java/util/List.html)可以包含重复元素的有序集合，它可以使用下标访问集合里面的元素，`List`也被称为顺序表

## 接口
```java
public interface List<E> extends Collection<E> {

    // 往集合里面添加一个元素，添加到集合的最后面
    boolean add(E e);
    
    // 获取下标为index的元素，List的下标和数组一样，从0开始
    E get(int index);

    // 替换下标为index的元素为指定的元素
    E set(int index, E element);

    // 添加一个元素到下标为index的位置，接着原来这个位置及其后面的元素都向后移动
    // 如果index越界了，则插入失败，并抛出异常，所以index不能大于集合的元素个数
    void add(int index, E element);

    // 移除下标为index的位置的元素，接着把这个位置后面的元素向前移动
    // 如果index越界了，则移除失败，并抛出异常，所以index不能大于或等于集合的元素个数
    E remove(int index);

    // 查找集合中对应的元素，找到了则返回其下标，找不到返回-1
    // 如果对应元素在集合里面出现了多次，将会返回最早出现（最小下标）的元素的下标
    int indexOf(Object o);

    // 查找集合中对应的元素，找到了则返回其下标，找不到返回-1
    // 如果对应元素在集合里面出现了多次，将返回最后出现（最大下标）的元素的下标
    int lastIndexOf(Object o);

    // 返回有序的迭代器
    ListIterator<E> listIterator();

    // 返回从指定下标开始的有序的迭代器
    ListIterator<E> listIterator(int index);
    
    // 返回集合的一部分元素，从fromIndex开始（包括fromIndex），到toIndex结束（不包括toIndex）
    List<E> subList(int fromIndex, int toIndex);
}
```

`List`接口定义的操作可以分成4类：

### 位置访问
>__Positional access__ — manipulates elements based on their numerical position in the list. This includes methods such as get, set, add, addAll, and remove.

* E	get(int index)

* E	set(int index, E element)

* void add(int index, E element)

* boolean addAll(int index, Collection<? extends E> c)

* E	remove(int index)

### 搜索
>__Search__ — searches for a specified object in the list and returns its numerical position. Search methods include indexOf and lastIndexOf.

* int indexOf(Object o)

* int lastIndexOf(Object o)

### 迭代
>__Iteration__ — extends Iterator semantics to take advantage of the list's sequential nature. The listIterator methods provide this behavior.

* Iterator<E> iterator()

* ListIterator<E> listIterator()

* ListIterator<E> listIterator(int index)

### 范围视图
>__Range-view__ — The sublist method performs arbitrary range operations on the list.

* List<E> subList(int fromIndex, int toIndex)

## 非线程安全的实现类

### ArrayList

[ArrayList]https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html

* 构造器
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

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
}
```

* 默认值
默认容量为10，但是这个默认容量不是创建实例的时候使用的，而是在第一次扩容的时候使用的，比如当使用了无参构造器创建了`ArrayList`实例后，调用add方法添加元素的时候，将会使用到这个默认容量。`ArrayList`每次扩容的时候容量增长__1.5倍__
```java
    // 当在构造器指定的容量值为0的时候，将使用这个值
    private static final Object[] EMPTY_ELEMENTDATA = {};
    // 当使用无参构造器的时候，会使用这个值
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    private static final int DEFAULT_CAPACITY = 10;
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

* 底层存储结构
底层存储结构为__数组__
```java
transient Object[] elementData;
```

### LinkedList

[LinkedList](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedList.html)

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

* 底层存储结构
底层存储结构为__链表__
```java
    // 头节点
    transient Node<E> first;
    // 尾节点
    transient Node<E> last;

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


## 线程安全的实现类

### Vector

[Vector](https://docs.oracle.com/javase/8/docs/api/java/util/Vector.html)

* 构造器
```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    public Vector(int initialCapacity, int capacityIncrement) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        this.elementData = new Object[initialCapacity];
        this.capacityIncrement = capacityIncrement;
    }

    public Vector(int initialCapacity) {
        this(initialCapacity, 0);
    }

    public Vector() {
        this(10);
    }

    public Vector(Collection<? extends E> c) {
        elementData = c.toArray();
        elementCount = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
    }
}
```

* 默认值
initialCapacity（默认容量）为10，使用无参构造器的时候将创建一个大小为默认容量的数组。
capacityIncrement没有指定的时候默认为0，该变量是扩容的时候要增长的值，如果为0，则扩容的时候容量增长__2倍__，如果不为0，则容量增长为旧容量 + capacityIncrement
```java
    protected int capacityIncrement;
```

* 底层存储结构
底层存储结构为__数组__
```java
protected Object[] elementData;
```

### CopyOnWriteArrayList

[CopyOnWriteArrayList](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CopyOnWriteArrayList.html)

* 构造器
```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }

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

    public CopyOnWriteArrayList(E[] toCopyIn) {
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }
}
```

* 默认值
当使用无参构造函数的时候，array变量的默认值为`new Object[0]`
```java
    /** The lock protecting all mutators */
    final transient ReentrantLock lock = new ReentrantLock();
    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```

* 底层存储结构
底层存储结构为数组。需要注意的是，该类在访问集合的元素的时候不需要加锁，因此读取效率较高，在修改元素的时候需要加锁，在修改元素的时候（添加，移除，替换），不但要加锁，而且要创建一个新的数组并把原来的元素拷贝过去新的数组，所以写入效率较低，仅适用于读多写少的场景
```java
    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```

