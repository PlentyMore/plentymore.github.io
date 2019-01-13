---
title: Java集合框架
date: 2019-01-08 21:12:02
tags:
    - JavaBasic
    - Collection
---

Java集合框架（Java Collections Framework）是用来操纵和表示集合的统一框架，该框架包含三个要素：接口，实现，算法

## 接口

![c](https://docs.oracle.com/javase/tutorial/figures/collections/colls-coreInterfaces.gif)

Java集合框架里面的接口是分级的，具体如上图，[Collection](https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html)是最顶层的接口，它定义了最基本的集合操作，`Collection`有4个子接口，分别为[Set](https://docs.oracle.com/javase/8/docs/api/java/util/Set.html)，[List](https://docs.oracle.com/javase/8/docs/api/java/util/List.html)，[Queue](https://docs.oracle.com/javase/8/docs/api/java/util/Queue.html)，[Deque](https://docs.oracle.com/javase/8/docs/api/java/util/Deque.html)。[Map](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html)不是真正的`Collection`，它不是`Collection`的子接口，但是它是Java集合框架的一部分。

### Collection

[Collection](https://docs.oracle.com/javase/8/docs/api/java/util/Collection.html)是Java集合框架最顶层的接口，`Collection`表示由一系列对象组成的一个逻辑单元，这些对象称为元素，所以一个集合由元素组成，这个接口定义了最通用的集合操作。

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

#### 查询操作

* int size()

* boolean isEmpty()

* boolean contains(Object o)

#### 批量操作

* boolean containsAll(Collection<?> c)

* boolean addAll(Collection<? extends E> c)

* boolean removeAll(Collection<?> c)
    
* boolean retainAll(Collection<?> c)

* void clear()

#### 数组操作

* Object[] toArray()

* <T> T[] toArray(T[] a)

#### 聚合操作

聚合操作（Aggregate Operations）在JDK1.8以后才支持

* default Stream<E> stream()

* default Stream<E> parallelStream() 该方法获取的stream能够充分利用CPU多核心提升性能

#### 迭代操作

有3中方式可以对`Collection`进行迭代操作（遍历集合里面的元素）

* 使用聚集操作 （JDK1.8+）
```java
al.stream().forEach(e->System.out.println(e));
```

* 使用迭代器
```java
// 迭代器的接口如下
public interface Iterator<E> {
    boolean hasNext();
    E next();
    void remove(); //optional
}
// 具体的使用例子如下
static void filter(Collection<?> c) {
	// 调用Collection的iterator方法获取迭代器实例
	// 然后调用迭代器的next方法访问元素
    for (Iterator<?> it = c.iterator(); it.hasNext(); )
        if (!cond(it.next()))
            it.remove();
}
```

* 使用for-each循环
```java
for (Object o : collection)
    System.out.println(o);
```

### Set
[Set](https://docs.oracle.com/javase/8/docs/api/java/util/Set.html)拓展了`Collection`接口，表示不包含重复元素的集合。

它没有引入更多的方法，不过它从`Collection`接口继承的某些方法有属于它自身的特定含义，比如add方法，在`Set`中，add方法也是往集合里面添加元素，不过如果集合里面已经有相同的元素了，则添加失败。

```java
public interface Set<E> extends Collection<E> {
    // 往集合里面添加一个元素，如果元素已存在，则添加失败，返回false
    boolean add(E e);
}
```

`Set`的各种操作和`Collection`的一致

### List
[List](https://docs.oracle.com/javase/8/docs/api/java/util/List.html)拓展了`Collection`接口，它表示有序的集合，也被称为顺序表，它可以向数组一样使用下标访问集合里面的元素。
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

#### 位置访问操作

`List`可以使用下标来访问元素，比如下面的几个方法

* E get(int index)

* E set(int index, E element)

* void add(int index, E element)

* E remove(int index)

#### 查找操作

* int indexOf(Object o)

* int lastIndexOf(Object o)

#### 迭代操作

除了`Iterator`之外，`List`还有自己的`ListIterator`，`ListIterator`是有序的迭代器，使用这个迭代器迭代元素的时候不仅可以向后访问元素，还可以向前访问元素

* ListIterator<E> listIterator(int index)

* List<E> subList(int fromIndex, int toIndex)

```java
public interface ListIterator<E> extends Iterator<E> {
    boolean hasNext();
    E next();
    boolean hasPrevious();
    E previous();  // 访问前一个元素
    int nextIndex();
    int previousIndex();
    void remove();
    void set(E e);
    void add(E e);
}
```

#### 范围视图操作

* List<E> subList(int fromIndex, int toIndex)
使用例子：
```java
public static <E> List<E> dealHand(List<E> deck, int n) {
    int deckSize = deck.size();
    List<E> handView = deck.subList(deckSize - n, deckSize);
    List<E> hand = new ArrayList<E>(handView);
    handView.clear();
    return hand;
}
```

`List`的其它操作也和`Collectino`一致（因为是从`Collection`继承的）


### Queue
[Queue](https://docs.oracle.com/javase/8/docs/api/java/util/Queue.html)表示队列，队列是用来保存一系列的元素用于后续处理的（处理完就从队列中移除），在Java集合框架中，队列一般是先进先出（FIFO）的，但是也有不是先进先出的队列，比如优先级队列（`PriorityQueue`）。

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
可以看到`Queue`方法一般都提供两种形式，一种是抛出异常的（add，remove，element），一种是不抛出异常的（offer，poll，peek）。队列的主要操作有3种：

* 插入（Insert）__add__（抛异常），__offer__（返回false）

* 移除（Remove）__remove__（抛异常），__poll__（返回false）

* 检索（Examine/Retrieve）__element__（抛异常），__peek__（返回false）

### Deque
[Deque](https://docs.oracle.com/javase/8/docs/api/java/util/Deque.html)表示双向队列，既可以往队首添加元素，也有可以往队尾添加元素的队列。
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

双向队列的操作和队列基本上是一致的，只不过双向队列可以从两个方向进行操作，下面的表格总结了操作的方法，并展示了这些方法的不同之处

Type of Operation | First Element (Beginning of the Deque instance) | Last Element (End of the Deque instance)
-------------     | ---------- | ----------- | -----------
Insert | addFirst(e)<br>offerFirst(e) | addLast(e)<br>offerLast(e)
Remove | removeFirst()<br>pollFirst() | removeLast()<br>pollLast()
Examine | getFirst()<br>peekFirst() | getLast()<br>peekLast()

每个格子都有两个方法，前面方法会抛出异常，后面的方法会返回false

### Map
[Map](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html)用来表示键值对（key-value）映射，每个key对应一个value，`Map`是这些映射关系的集合，`Map`包含多个键值对，不过每个键（key）必须是唯一的，不能是相同的。`Map`不是`Collection`，但它属于集合框架

```java
public interface Map<K,V> {
    // Query Operations

    // Map中的键值对数量
    int size();

    // Map是否为空，即没有键值对
    boolean isEmpty();

    // Map中是否包含特定的键
    boolean containsKey(Object key);

    // Map中是否包含特定的值
    boolean containsValue(Object value);

    // 根据key获取对应的value，如果key不存在则返回null
    V get(Object key);

    // Modification Operations

    // 向Map中放入一个键值对，返回键值对的值
    // 如果键在Map中已存在，则使用给定的值来更新原来的键值对的值，返回原来的旧值
    V put(K key, V value);

    // 根据key移除键值对，如果key在Map中不存在，则返回null，否则返回key对应的value
    V remove(Object key);

    // Bulk Operations

    // 把另一个Map的键值对放入该Map，向当于将另一个Map的键值对调用put方法放到该Map中
    void putAll(Map<? extends K, ? extends V> m);

    // 移除Map中所有的键值对
    void clear();

    // Views

    // 返回该Map中所有键的集合，返回类型为Set，因为键都是不相同的
    Set<K> keySet();

    // 返回该Map中的所有值的集合，返回类型为Colleectino
    Collection<V> values();

    // 返回该Map中所有键值对的集合，返回类型为Set<Map.Entry<K, V>>
    // Map.Entry表示一个键值对
    Set<Map.Entry<K, V>> entrySet();

    // 表示一个键值对
    interface Entry<K,V> {
        // 返回该键值对的key
        K getKey();

        // 返回该键值对的value
        V getValue();

        // 修改该键值对的value
        V setValue(V value);

        // 判断该键值对与指定的键值对是否相等
        boolean equals(Object o);

        int hashCode();

        // 省略了部分方法
    }

    // Comparison and hashing

    boolean equals(Object o);

    int hashCode();
}
```

#### 基本操作

* V get(Object key) 获取key对应的value

* V put(K key, V value) 放入一个key-value（称为Entry）

* V remove(Object key) 移除一个key-value

* boolean containsKey(Object key) Map中是否包含特定的key

* boolean containsValue(Object value) Map中是否包含特定的value

#### 集合视图

* Set<K> keySet() 返回Map中的所有key的集合（没有重复元素，所以是Set）

* Collection<V> values() 返回Map的所有value的集合

* Set<Map.Entry<K, V>> entrySet() 返回Map中的所有key-value的集合（Map中每个key-value都是唯一的，所以没有重复元素，因此返回Set）

#### 批量操作

* void putAll(Map<? extends K, ? extends V> m) 如果另一个Map中有和该Map相同的key，则更新该Map中的value

* void clear()

#### Multimaps

Multimaps是一种特殊的`Map`，它的一个key可以对应一个或者多个value，具体实现起来也不难，直接把`List`作为value就可以了，比如`Map<String, List<String>>`，`List`可以有多个元素，所以相当于一个key对应多个value


## 实现
Java集合框架的实现可以分为6种类型：

### General-purpose implementations
>__General-purpose implementations__ are the most commonly used implementations, designed for everyday use. They are summarized in the table titled General-purpose-implementations.

一般用途的实现，比如`ArrayList`，`HashMap`

### Special-purpose implementations
>__Special-purpose implementations__ are designed for use in special situations and display nonstandard performance characteristics, usage restrictions, or behavior.

特殊用途的实现，比如`WeakHashMap`，`JumboEnumSet`，`EnumSet`

### Concurrent implementations
>__Concurrent implementations__ are designed to support high concurrency, typically at the expense of single-threaded performance. These implementations are part of the java.util.concurrent package.

用于并发场景的实现，比如`ConcurrentHashMap`，`CopyOnWriteArrayList`，在java.util.concurrent包下

### Wrapper implementations
>__Wrapper implementations__ are used in combination with other types of implementations, often the general-purpose ones, to provide added or restricted functionality.

包装器实现，比如`UnmodifiableCollection`，`UnmodifiableSortedSet`，`SynchronizedCollection`，大部分都是`Collections`的静态内部类

### Convenience implementations
>__Convenience implementations__ are mini-implementations, typically made available via static factory methods, that provide convenient, efficient alternatives to general-purpose implementations for special collections (for example, singleton sets).

便捷实现，比如`EmptySet`，`EmptyList`，`SingletonSet`，大部分是`Collections`的静态内部类

### Abstract implementations
>__Abstract implementations__ are skeletal implementations that facilitate the construction of custom implementations — described later in the Custom Collection Implementations section. An advanced topic, it's not particularly difficult, but relatively few people will need to do it.

抽象实现，比如`AbstractCollection`，`AbstractList`，基于抽象实现类可以快速地实现自己的集合类

下面的表格给出了集合框架的用作一般用途的实现类
__General-purpose Implementations__

| Interfaces | Hash table Implementations | Resizable array Implementations | Tree Implementations | Linked list Implementations | Hash table + Linked list Implementations |
| ---------- | ----------- | ---------- | ----------- | ------------- | ------------ |
| Set        | HashSet	   | 	&nbsp;  | TreeSet     | &nbsp;  |  LinkedHashSet   |
| List       | &nbsp;      | ArrayList  |  &nbsp;     |    LinkedList    |   &nbsp;   | 
| Queue	 	 |  &nbsp;     |  &nbsp;    |   &nbsp;    |  &nbsp;   |    &nbsp;     |
| Deque	     | &nbsp; 	   | ArrayDeque |    &nbsp;   |   LinkedList     |  &nbsp;    |
| Map        | HashMap	   | &nbsp;     | TreeMap     |  &nbsp;     |   LinkedHashMap |


## 算法
集合框架实现的算法有：排序（Sorting），洗牌(Shuffling)，常规数据操作(Routine Data Manipulation)，查找(Searching)，组合(Composition)，查找极值（Finding Extreme Values）

### Sorting
Java集合框架实现的排序算法通常接收一个`List`作为参数，按照自然顺序进行排序，比如下面的例子
```java
public class Sort {
    public static void main(String[] args) {
        List<String> list = Arrays.asList(args);
        Collections.sort(list);
        System.out.println(list);
    }
}
```
使用的排序算法为优化的归并排序算法，除了按照自然顺序进行排序之外，也可以使用指定的`Comparator `进行排序，比如下面的例子：
```java
// Make a List of all anagram groups above size threshold.
List<List<String>> winners = new ArrayList<List<String>>();
for (List<String> l : m.values())
    if (l.size() >= minGroupSize)
        winners.add(l);

// Sort anagram groups according to size
Collections.sort(winners, new Comparator<List<String>>() {
    public int compare(List<String> o1, List<String> o2) {
        return o2.size() - o1.size();
    }});

// Print anagram groups.
for (List<String> l : winners)
    System.out.println(l.size() + ": " + l);
```

### Shuffling
洗牌算法和排序算法是对立的，排序算法是使得`List`里面的元素有序，而洗牌算法是打乱元素的顺序。
这种算法在Java集合框架有两种形式，第一种是接收一个`List`作为参数，然后使用默认的随机算法把元素顺序打乱，比如下面的例子：
```java
public class Shuffle {
    public static void main(String[] args) {
        List<String> list = Arrays.asList(args);
        Collections.shuffle(list);
        System.out.println(list);
    }
}
```

第二种是自己提供一个`Random`对象，并使用这个对象的随机逻辑来把元素顺序打乱，比如下面的例子：
```java
public class Shuffle {
    public static void main(String[] args) {
        List<String> list = new ArrayList<String>();
        for (String a : args)
            list.add(a);
        Collections.shuffle(list, new Random());
        System.out.println(list);
    }
}
```

### Routine Data Manipulation
集合框架提供以下的5个方法来对`List`对象进行常规的数据操作：
* __reverse__ — reverses the order of the elements in a List.
* __fill__ — overwrites every element in a List with the specified value. This operation is useful for reinitializing a List.
* __copy__ — takes two arguments, a destination List and a source List, and copies the elements of the source into the destination, overwriting its contents. The destination List must be at least as long as the source. If it is longer, the remaining elements in the destination List are unaffected.
* __swap__ — swaps the elements at the specified positions in a List.
* __addAll__ — adds all the specified elements to a Collection. The elements to be added may be specified individually or as an array.

具体实现在[Collections](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html)类

### Searching
集合框架使用二分查找算法来查找有序的`List`中特定的元素，这个方法有两张形式，第一种是接收一个`List`类型的参数和一个key（要查找的元素），这种形式假定`List`是按照自然顺序升序排序的，然后使用二分查找算法进行查找，例子如下：
```java
int pos = Collections.binarySearch(list, key);
```

第二种形式是自己提供一个`Comparator`，例子如下：
```java
int pos = Collections.binarySearch(list, key, new Comparator(){...});
```

### Composition
* __frequency__ — counts the number of times the specified element occurs in the specified collection
* __disjoint__ — determines whether two Collections are disjoint; that is, whether they contain no elements in common

在[Collectins](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html)类里面可以看到这两个方法的实现
```java
// Returns the number of elements in the specified collection equal to the specified object.
static int	frequency(Collection<?> c, Object o)

// Returns true if the two specified collections have no elements in common.
static boolean	disjoint(Collection<?> c1, Collection<?> c2)

```

### Finding Extreme Values
查找集合中的最大或者最小值，同样的，也是提供两种形式，第一种是按照自然顺序，第二种是自己提供`Comparator`，具体实现也是在[Collectinos](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html#max-java.util.Collection-)类
```java
// Returns the maximum element of the given collection, 
// according to the natural ordering of its elements
static <T extends Object & Comparable<? super T>> T max(Collection<? extends T> coll)

// Returns the maximum element of the given collection, 
// according to the order induced by the specified comparator
static <T> T max(Collection<? extends T> coll, Comparator<? super T> comp)

// Returns the minimum element of the given collection, 
// according to the natural ordering of its elements
static <T extends Object & Comparable<? super T>> T min(Collection<? extends T> coll)

// Returns the minimum element of the given collection, 
// according to the order induced by the specified comparator
static <T> T min(Collection<? extends T> coll, Comparator<? super T> comp)
```

笔记来源： [Collections]（https://docs.oracle.com/javase/tutorial/collections/index.html)