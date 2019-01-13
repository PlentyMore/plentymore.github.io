---
title: Java Set接口常用实现类
date: 2019-01-09 18:56:29
tags:
    - Collection
---

[Set](https://docs.oracle.com/javase/8/docs/api/java/util/Set.html)拓展了`Collection`接口，表示不包含重复元素的集合。

## 接口

`Set`的方法基本上全是从`Collection`继承的，没有增加新的方法，不过因为`Set`不包含重复元素，所以它的add方法有自己的含义（当集合中已经有相同的元素时，则添加失败）。
```java
public interface Collection<E> extends Iterable<E> {
    // Query Operations（查询操作）
    // 返回集合的元素个数
    int size();

    // 集合是否为空，即没有元素
    boolean isEmpty();

    // 查看集合是否包含特定的元素
    boolean contains(Object o);

    // 返回集合的迭代器，用于遍历集合元素
    Iterator<E> iterator();

    // 把集合里面的全部元素转换成数组类型返回
    Object[] toArray();

    // 和上面的方法类似，不过这里可以指定元素的类型
    <T> T[] toArray(T[] a);

    // Modification Operations（修改操作）
    // 添加一个元素到集合中
    boolean add(E e);

    // 从集合中移除特定的元素
    boolean remove(Object o);

    // Bulk Operations（批量操作）
    // 集合是否包含另一个集合的所以元素
    boolean containsAll(Collection<?> c);

    // 把另一个集合的元素添加到该集合
    boolean addAll(Collection<? extends E> c);
 
    // 移除该集合中和另一个集合相同的元素
    boolean removeAll(Collection<?> c);
    
    // 只保留该集合中和另一个集合相同的元素
    boolean retainAll(Collection<?> c);

    // 清空集合中的元素
    void clear();

    // Comparison and hashing

    boolean equals(Object o);

    int hashCode();
}
```
上面直接贴`Collection`接口的方法，因为`Set`没有添加其它的方法。

## 非线程安全的实现类

### HashSet

[HashSet](https://docs.oracle.com/javase/8/docs/api/java/util/HashSet.html)

* 构造器
```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    public HashSet() {
        map = new HashMap<>();
    }

    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
}
```

* 默认值
PRESEENT用来作为key-value的值
```java
    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
```

* 底层存储结构
它利用了`HashMap`，所以底层存储结构和`HashMap`一样，实际上`HashSet`是只使用到`HashMap`的key的`HashMap`，它添加元素的时候会把要添加的元素作为key，然后PRESENT作为value，key是非重复的
```java
    private transient HashMap<E,Object> map;
```

### TreeSet

[TreeSet](https://docs.oracle.com/javase/8/docs/api/java/util/TreeSet.html)是有序的Set，里面的元素按照自然顺序或者指定的`Comparator`进行排序

* 构造器
```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
}
```

* 默认值
可以看到这里和`HashSet`是类似的，也是有一个PRESENT
```java
    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
```

* 底层存储结构
它利用了`TreeMap`，所以底层存储结构和`TreeMap`（`TreeMap`实现了`NavigableMap`）一样，`TreeSet`在添加元素的时候，把要添加的元素作为key，然后PRESENT作为value添加到`TreeMap`中，key是非重复的
```java
    private transient NavigableMap<E,Object> m;
```

### LinkedHashSet

[LinkedHashSet](https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashSet.html)是有序的

* 构造器
```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    public LinkedHashSet() {
        super(16, .75f, true);
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }
}
```

* 默认值
无

* 底层存储结构
实际上它利用了`LinkedHashMap`，所以底层存储结构和`LinkedHashMap`一样

## 线程安全的实现类

### CopyOnWriteArraySet
[CopyOnWriteArraySet](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CopyOnWriteArraySet.html)

* 构造器
```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E>
        implements java.io.Serializable {
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }

    public CopyOnWriteArraySet(Collection<? extends E> c) {
        if (c.getClass() == CopyOnWriteArraySet.class) {
            @SuppressWarnings("unchecked") CopyOnWriteArraySet<E> cc =
                (CopyOnWriteArraySet<E>)c;
            al = new CopyOnWriteArrayList<E>(cc.al);
        }
        else {
            al = new CopyOnWriteArrayList<E>();
            al.addAllAbsent(c);
        }
    }
}
```

* 默认值
无

* 底层存储结构
利用了`CopyOnWriteArrayList`，所以其底层存储结构是数组。该类除了不包含重复元素之外，其它的特性都和`CopyOnWriteArrayList`一样
```java
    private final CopyOnWriteArrayList<E> al;
```

### ConcurrentSkipListSet

[ConcurrentSkipListSet](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentSkipListMap.html)是有序的，按照自然顺序或者指定的`Comparator`排序。

* 构造器
```java
public class ConcurrentSkipListSet<E>
    extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {
    public ConcurrentSkipListSet() {
        m = new ConcurrentSkipListMap<E,Object>();
    }

    public ConcurrentSkipListSet(Comparator<? super E> comparator) {
        m = new ConcurrentSkipListMap<E,Object>(comparator);
    }

    public ConcurrentSkipListSet(Collection<? extends E> c) {
        m = new ConcurrentSkipListMap<E,Object>();
        addAll(c);
    }

    public ConcurrentSkipListSet(SortedSet<E> s) {
        m = new ConcurrentSkipListMap<E,Object>(s.comparator());
        addAll(s);
    }

    /**
     * For use by submaps
     */
    ConcurrentSkipListSet(ConcurrentNavigableMap<E,Object> m) {
        this.m = m;
    }
}
```

* 默认值
无

* 底层存储结构
利用了`ConcurrentSkipListMap`（实现了`ConcurrentNavigableMap`接口），所以底层结构和它一样，是链表
```java
    static final class Node<K,V> {
        final K key;
        volatile Object value;
        volatile Node<K,V> next;
    }
```

