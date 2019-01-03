---
title: Hashtable
date: 2018-08-15 21:13:14
header-img: https://i.imgur.com/RAwKwAj.jpg
cdn: 'header-off'
tags:
    - Container
---

{% meting "26259014" "netease" "song" %}

## Hashtable的结构

和HashMap类似，Hashtable主要由一张哈希表构成，这张表由Entry<?,?>类型的数组构成，
它和HashMap的主要却别是，它是线程安全的，且不能添加null键值对。在早期的JDK版本，
Hashtable只继承了Dictionary类，在JDK 1.2之后它实现了Map接口，成为了Collections Framework的一部分。
Hashtable可以算是历史遗留类，它的实现比较简单，通过同步方法来保证线程安全，但效率比较低，因为线程只能互斥的调用同步方法，
就是说同一时刻只有一个线程能读取或者放入键值对。

## Hashtable的静态成员变量
```java
/**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

```
### MAX_ARRAY_SIZE
Hashtable的哈希表的最大容量，即数组table的最大长度，值为Integer.MAX-8。

## Hashtable的实例域
```java
    /**
     * The hash table data.
     */
    private transient Entry<?,?>[] table;

    /**
     * The total number of entries in the hash table.
     */
    private transient int count;

    /**
     * The table is rehashed when its size exceeds this threshold.  (The
     * value of this field is (int)(capacity * loadFactor).)
     *
     * @serial
     */
    private int threshold;

    /**
     * The load factor for the hashtable.
     *
     * @serial
     */
    private float loadFactor;

    /**
     * The number of times this Hashtable has been structurally modified
     * Structural modifications are those that change the number of entries in
     * the Hashtable or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the Hashtable fail-fast.  (See ConcurrentModificationException).
     */
    private transient int modCount = 0;
    
// Views
    
        /**
         * Each of these fields are initialized to contain an instance of the
         * appropriate view the first time this view is requested.  The views are
         * stateless, so there's no reason to create more than one of each.
         */
        private transient volatile Set<K> keySet;
        private transient volatile Set<Map.Entry<K,V>> entrySet;
        private transient volatile Collection<V> values;    
    
```

### table

`Entry<?,?>[] table`

Entry类型的数组，代表一张哈希表，Entry的具体结构如下：
```java
/**
     * Hashtable bucket collision list entry
     */
    private static class Entry<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Entry<K,V> next;

        protected Entry(int hash, K key, V value, Entry<K,V> next) {
            this.hash = hash;
            this.key =  key;
            this.value = value;
            this.next = next;
        }
        ...........................其他方法已省略
}
```
可以看到Entry的结构和HashMap.Node的结构是一样的，实例域都是hash，key，value，next这4个

### count
int类型，表中的键值对的个数

### threshold
int类型，哈希表的阈值，当表中的键值对个数超过这个值的时候，就要对表进行扩容。计算公式为：threshold = (int)(capacity * loadFactor)

### loadFactor
float类型，哈希表的负载因子，表示使用的数组元素个数最多能达到数组长度的百分比。
比如数组长度为16，负载因子为0.75，则最多能使用该数组的16 * 0.75 = 12个元素

### modCount
int类型，哈希表被修改的次数，比如放入元素、移除元素都算一次修改，用于实现fail-fast机制。

### keySet
`Set<K> keySet`，键的集合

### entrySet
`Set<Map.Entry<K,V>>` 键值对的集合
 
### values
`Collection<V>` 值的集合    
    
## Hashtable的构造方法
Hashtable有4个构造方法

```java
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
            initialCapacity = 1;  //这里可以看到最小的初始容量为1
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];  //这里直接就为table分配了空间
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
        //可以自己指定初始容量
    }

    /**
     * Constructs a new, empty hashtable with a default initial capacity (11)
     * and load factor (0.75).
     */
    public Hashtable() {
        this(11, 0.75f);
        //不提供任何参数，则初始容量为11，load factor为0.75
        //所以11为默认的容量，0.75为默认的负载因子
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
        //如果t的元素个数的2被小于11，则初始容量仍为11
        this(Math.max(2*t.size(), 11), 0.75f);
        putAll(t);
    }
```

## put
```java
    /**
     * Maps the specified <code>key</code> to the specified
     * <code>value</code> in this hashtable. Neither the key nor the
     * value can be <code>null</code>. <p>
     *
     * The value can be retrieved by calling the <code>get</code> method
     * with a key that is equal to the original key.
     *
     * @param      key     the hashtable key
     * @param      value   the value
     * @return     the previous value of the specified key in this hashtable,
     *             or <code>null</code> if it did not have one
     * @exception  NullPointerException  if the key or value is
     *               <code>null</code>
     * @see     Object#equals(Object)
     * @see     #get(Object)
     */
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {  //1
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();  //2
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {  //3
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
        addEntry(hash, key, value, index);
        return null;
    }
```
可以看到这个方法有synchronized关键字修饰，是同步方法。这个方法的主要步骤如下：

1. 检查键值对的值是否为null，如果是则抛出异常。这里没有检测键是否为空，因此当我们可以传一个为null的键进去时，
后面的`key.hashCode()`就会抛出空指针异常，因为key为null，所以这个Hashtable的设计是有问题的。
由于key和value为空都会使该方法抛出异常，因此不能往Hashtable里面添加null键值对。

2. 调用key的hashCode方法计算hash，再用hash和0x7FFFFFFF做与运算，得到的结果和table.length取余，
最后得到该键值对的在数组中的索引，再利用索引取出里面的元素。

3. 如果通过上面计算出的索引取出的元素不为空，说明该索引的位置已经存储了元素，接下来对该元素的所在的链表进行遍历，
遍历过程中寻找与要插入的键值对的键相等的元素，如果找到这样的元素，则直接更新这个元素的值，接着返回这个元素更新前的值。
如果没有这样的元素，则调用addEntry方法将键值对插入。

接下来看看addEntry方法的实现：
```java
private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry<?,?> tab[] = table;
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```
1. 该方法首先使modCount错的值自增1

2. 判断当前的元素个数是否大于等于阈值（threshold），如果是，则调用rehash()扩容，扩容后，
使用扩容后的数组长度重新计算要插入的键值对所对应的数组索引 

3. 为新的键值对创建一个新的Entry对象，把该对象的next指向该数组位置原先的元素（如果元素存在的话），
然后把该对象放到上面计算出的数组索引位置上，count自增1，即Hashtable的元素个数增加1。
从这里可以看出，当插入一个键值对时，如果要插入的位置已经有元素了，则使新插入的键值对的next变量指向这个元素，
因此Hashtable的插入是在链表的头部插入，而HashMap是在尾部。

rehash方法的实现如下：
```java
    /**
     * Increases the capacity of and internally reorganizes this
     * hashtable, in order to accommodate and access its entries more
     * efficiently.  This method is called automatically when the
     * number of keys in the hashtable exceeds this hashtable's capacity
     * and load factor.
     */
    @SuppressWarnings("unchecked")
    protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {  //遍历旧数组的所有元素
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {  //遍历链表
                Entry<K,V> e = old;  //将当前遍历到的元素存储到e
                old = old.next;  //old指向它的下一个元素

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;  //计算元素在新空间的索引
                e.next = (Entry<K,V>)newMap[index];  //使当前遍历到的元素的next指向链表的头部
                newMap[index] = e;  //让当前遍历到的元素变成链表的头部
            }
        }
    }
```

1. 将当前的table存储到oldMap变量中，当前的table的长度存储到oldCapacity变量中

2. 将新容量设置为(oldCapacity << 1) + 1，存储到newCapacity变量中，如果新容量大于MAX_ARRAY_SIZE，
则继续判断oldCapacity是否已经达到了最大值（oldCapacity == MAX_ARRAY_SIZE），如果是，则不进行扩容，直接返回。
如果不是，则将新容量设置为最大值

3. 为table分配新的空间，长度为newCapacity，使modCount自增1，重新计算threshold

4. 遍历旧数组的所有元素，计算元素在新空间上的索引，将这些元素移动到新的空间上，移动之后，原来的链表上的元素位置顺序会发生反转，
比如原来是table[x]->1->2->3，移动之后会变成table[y]->3->2->1

## get
```java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key.equals(k))},
     * then this method returns {@code v}; otherwise it returns
     * {@code null}.  (There can be at most one such mapping.)
     *
     * @param key the key whose associated value is to be returned
     * @return the value to which the specified key is mapped, or
     *         {@code null} if this map contains no mapping for the key
     * @throws NullPointerException if the specified key is null
     * @see     #put(Object, Object)
     */
    @SuppressWarnings("unchecked")
    public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();  //计算key的hash
        int index = (hash & 0x7FFFFFFF) % tab.length;  //计算索引
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {  //遍历链表
            if ((e.hash == hash) && e.key.equals(key)) {  //key相等则返回它的value
                return (V)e.value;
            }
        }
        return null;
    }
```

同样的，get方法也不能传入null，会引发空指针异常。它的实现也非常简单，直接根据hash计算索引，
取出索引位置的元素，从这个元素开始遍历它所在的链表，直到找到key相等的元素，和HashMap相比，
没有了从红黑树中查找元素的过程。

## remove
```java
    /**
     * Removes the key (and its corresponding value) from this
     * hashtable. This method does nothing if the key is not in the hashtable.
     *
     * @param   key   the key that needs to be removed
     * @return  the value to which the key had been mapped in this hashtable,
     *          or <code>null</code> if the key did not have a mapping
     * @throws  NullPointerException  if the key is <code>null</code>
     */
    public synchronized V remove(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {  //查找相同的key
                modCount++;
                if (prev != null) {  //prev记录了前一个元素
                    prev.next = e.next;  //pref的next指向要删除的元素的下一个元素
                } else {
                    tab[index] = e.next;  //prev为空，说明该要删除的元素在链表头部
                }
                count--;
                V oldValue = e.value;
                e.value = null;
                return oldValue;
            }
        }
        return null;
    }
```

remove方法的实现也比较简单，首先查找key，找到相同的key后就进行删除元素的操作。

## clear
```java
    /**
     * Clears this hashtable so that it contains no keys.
     */
    public synchronized void clear() {
        Entry<?,?> tab[] = table;
        modCount++;
        for (int index = tab.length; --index >= 0; )
            tab[index] = null;
        count = 0;
    }
```
该方法将清除table数组上的所有元素的引用，并将count设置为0