---
title: HashMap
date: 2018-08-14 12:42:36
header-img: https://i.imgur.com/RAwKwAj.jpg
cdn: 'header-off'
tags:
    - Container
---

{% meting "518863914" "netease" "song" %}
    
## HashMap的结构（JDK1.8）

HashMap主要由一张哈希表组成，这张表是一个Node<K,V>类型的数组，默认初始容量为16，每一个数组元素又被成为一个桶，
桶可以存放链表和其他结构（如TreeNode)。当链表过长（哈希冲突严重）时，链表将转换成其他结构（红黑树）。
需要注意的是，当表的容量（桶的数量）小于64的时候，即使链表长度大于8，也不会立刻转成红黑树，
而是对表进行扩容，在容量足够大（至少为64）之后，在长度大于等于8的链表上再次发生哈希冲突时才会将链表转换成红黑树。
因此链表转换成红黑树有两个条件:容量(桶的数量)大于等于64和链表长度大于等于8。

一般情况下的HashMap结构如下图：

![HashMap结构](https://i.imgur.com/3JylisY.png)

哈希冲突严重时的HashMap结构如下图：

![冲突严重时HashMap结构](https://i.imgur.com/6Je0d2K.png)

从上面的图可以看到Node转换成了TreeNode，即所谓的红黑树结构。

## HashMap的静态成员变量
```angularjs
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
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
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

```
### DEFAULT_INITIAL_CAPACITY
表的默认容量，值为16，取值必须为2的冥次方，没有显式指定时将使用该值

### MAXIMUM_CAPACITY
表的最大容量，值为1 << 30，即1073741824

### DEFAULT_LOAD_FACTOR
默认负载因子，值为0.75，当没有在显式指定时将使用该值

### TREEIFY_THRESHOLD
将链表转换为红黑树链表长度需要达到的最小值，默认值为8

### UNTREEIFY_THRESHOLD
某个桶上红黑树的节点个数逐渐减少到该值时，将红黑树转换成链表，默认值为6

### MIN_TREEIFY_CAPACITY
将链表转换成红黑树时表容量（桶数量）的最小值，该值为64，就是说只有容量大于等于64时才有可能会将链表转换成红黑树，__仅仅是链表长度大于8是不会将链表转换成红黑树的__。

## HashMap的实例域
```angularjs
/* ---------------- Fields -------------- */

    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;

    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * The number of key-value mappings contained in this map.
     */
    transient int size;

    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
    transient int modCount;

    /**
     * The next size value at which to resize (capacity * load factor).
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    int threshold;

    /**
     * The load factor for the hash table.
     *
     * @serial
     */
    final float loadFactor;
```
### table
 `Node<K,V>[] table`
 
 Node类型的数组，代表一张哈希表，table数组的每一个元素代表一个桶，数组长度长度即桶的个数，可以存放Node类型和TreeNode类型的元素。
 
 Node为HashMap的静态内部类，实现了Map.Map.Entry<K,V>接口，其具体实现如下：
 ```angularjs
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;  //该节点的哈希值
        final K key;  //键
        V value;  //值
        Node<K,V> next;  //链表当前节点的的下一个节点

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
        
        public final boolean equals(Object o) {
            //两个对象内存地址相同，则相等 
            if (o == this)
                return true;
            /*对象类型为Map.Entry及其子类，
            且两个对象对应的键和值都相等，则
            这两个对象相等*/
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

```
 
 TreeNode为HashMap的静态内部类，继承了LinkedHashMap.Entry<K,V>类，而LinkedHashMap.Entry<K,V>
 继承了HashMap.Node<K,V>类，即上面的Node类，因此实际上TreeNode为Node的子类，所以table数组也可以存放TreeNode类型的元素。
 
 ```angularjs
/**
     * Entry for Tree bins. Extends LinkedHashMap.Entry (which in turn
     * extends Node) so can be used as extension of either regular or
     * linked node.
     */
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        ...........................该类其他方法的实现在这里不列举了
}
```
 
 ### entrySet
 `Set<Map.Entry<K,V>> entrySet`
 
 键值对的集合，用于遍历HashMap的元素，调用其keySet()和values()方法可以分别得到单独的键的集合和值的集合
 
 ### size
 int类型，其值为表中存在元素的个数

### modCount
int类型，其值为该表被修改的次数，比如插入和移除元素都可以算一次修改，用于实现fail-fast机制

### threshold
int类型，其值为该表下一次扩容的阈值，当表的元素个数超过这个值时，将对表进行扩容，即为
table数组重新分配更大的空间。计算公式为：threshold = capacity * load factor。
如果table数组还没有被分配空间，即没有进行过扩容操作（resize）时，如果该域的值为0，
则它表示的是数组的初始容量没有指定，将使用默认的值进行扩容。当该域的值大于0时，
它存储的值是在构造器显式指定的initialCapacity的值，第一次为扩容（调用resize方法）时将使用这个值作为为table分配相应大小的空间。

### loadFactor
float类型，为该表的负载因子，表示该表的桶使用率最多可以达到多少。

## put
```angularjs
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}
```

put方法的实现接收一个键和一个值，该方法调用了putVal方法

```angularjs
/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)  //1
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)  //2
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))  //3
                e = p;
            else if (p instanceof TreeNode)  //4
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {  //5
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))  //6
                        break;
                    p = e;
                }
            }
            //7
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)  //8
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
如果成功添加了一个键值对，会使modCount自增1。从上面可以看出，往HashMap中放入一个键值对的大概步骤如下：

1. 判断table数组是否为空或者长度为0，如果是，则需要调用resize()方法为table数组分配空间，并将该方法返回的数组长度存储到变量n中

2. 根据Key的hash和table数组的长度计算Key对应的table元素的索引，若计算出的索引对应的元素为空，则创建该键值对的Node对象，并放到该索引位置的数组元素中

4. 如果上面计算出的索引位置已经有元素（不为null），则判断该元素的Key和要插入的元素的Key是否相等，相等则后面直接更新元素的值即可，不需要创建新的Node对象

5. 如果上面的元素的Key和要插入的元素的Key不相等，则判断该元素是否为TreeNode类型，如果是则使用TreeNode的putTreeVal方法将键值对放入HashMap

6. 如果上面的元素不是TreeNode类型，则直接遍历该元素的链表，如果遍历过程中找到其Key和要插入的Key相等的元素，则停止遍历。如果没有找到这样的元素，当遍历到最后一个元素后
（元素的next域为null）时，创建要插入的键值对的Node对象，并使next指向该对象，接着检查遍历次数是否大于等于8，如果是，则
调用treeifyBin方法，该方法可能会将链表转换为红黑树，也可能会对table数组进行扩容（当table数组长度小与64时候）。

7. 判断变量e是否为空，不为空说明存在存在一个元素，这个元素的Key与要插入的键值对的Key相同，此时将直接更新这个的元素的值为要插入键值对的值，然后返回这个元素未值更新前的值，方法调用结束。

8. 若上面的e为空，则使modCount和size的值自增1。e为空，则说明要创建一个新的Node对象，这需要检查插入新元素后的size是否大于threshold，如果大于，则需要进行扩容操作，调用resize方法。
最后调用afterNodeInsertion方法进行一些额外的操作（目前该方法没有做任何事情），返回null，方法调用结束。

接下来看看上面提到的一些其他的方法，比如resize方法，resize方法的实现如下：

```angularjs
/**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;  //1
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {  //2
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //3
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        //4
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {  //5
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;  //6
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];  //7
        table = newTab;
        if (oldTab != null) {  //8
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

resize方法的大概步骤如下：

1. 获取table数组当前长度，存储到oldCap变量中，threshold变量的值存储到oldThr变量中

2. 判断当前容量是否超过最大值，如果超过了，就不扩容了，直接返回就原table数组，如果没有超过，则尝试扩大容量为原来的2倍。具体操作：
如果oldCap > 0，则继续判断oldCap是否大于等于MAXIMUM_CAPACITY，即当前的table数组长度是否大于等于HashMap允许的最大值，如果是，
则将threshold的值设置为Integer.MAX_VALUE，然后返回原table数组，方法调用结束。如果不是，则检查oldCap*2的值是否
大于等于16并且小于MAXIMUM_CAPACITY，并将newCap的值设置为oldCap的2倍，如果是，则将newThr的值设置为oldCap的2倍（即原先的threshold的2倍）。

3. 如果oldCap <= 0，且oldThr > 0（threshold > 0），则将newCap的值设置为oldThr（因为当oldCap <= 0且threshold的值大于0时候，
threshold中存放的值是初始容量值，这个值是构造函数中指定的，可以在HashMap的构造器中看到这么一句`this.threshold = tableSizeFor(initialCapacity)`，
现在HashMap还没有为table数组分配空间，即没有进行过resize操作，而且在构造器中指定了默认容量，只不过这个默认容量保存在threshold变量中，
因此现在的新容量(newCap)应该为threshold中的值），此时真正的threshold值还没有计算出来。

4. 如果oldCap <=0 ，且oldThr <= 0，则newCap（新容量值）和newThr（新的阈值）均使用默认值。因为当oldThr等于0时，说明没有在构造器指定初始的容量值，所以使用
默认值。

5. 如果newThr为0，说明没有扩容（原因是原来的容量的2倍超过了最大值或者小于16），此时开始计算阈值（threshold）

6. 将上一步计算出的值保存到threshold变量中，该值就是扩容后的域值

7. 为table数组分配新的空间，数组长度为newCap

8. 如果table数组在resize之前有分配过的空间，则将分配过的空间存在的元素移动到新的空间上面。具体操作：
清除之前分配的空间上的所有引用，计算旧元素在新空间上的索引，将旧元素放到对应新空间的位置。


再看看treeifyBin方法，treeifyBin方法的实现如下：

```angularjs
    /**
     * Replaces all linked nodes in bin at index for given hash unless
     * table is too small, in which case resizes instead.  当表太小时只对表进行扩容
     */
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)  //1
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {  //2
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```

TreeifyBIn方法的主要操作如下：

1. 如果table数组还没有分配空间，或者table数组的长度小于64，则调用resize方法进行扩容
2. 否则将table数组上的指定的元素（调用treeifyBin方法时候会传入hash，通过这个hash可以计算出这个元素对应的索引，
该索引位置存放的元素为链表的第一个元素）所在的链表转换成红黑树。

## get
```angularjs
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

get方法接受一个任意类型的Key，该方法调用了getNode方法，getNode方法的实现如下：

```angularjs
/**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {  //1
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))  //2
                return first;
            if ((e = first.next) != null) {  //3
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {  //4
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;  //5
    }
```

该方法的主要步骤如下：

1. 判断表是否为空或者长度等于0，如果是则返回null，如果不是则根据Key的hash计算它的索引，
如过计算出的索引位置对应的元素为空，则返回null

2. 检查计算出的索引位置的第一个元素的Key和要查找的对象的Key是否相等，若相等则查找成功，返回第一个元素。

3. 检查第一个元素的next域是否为空，如果不为空，则检查next域所指向的对象是否为TreeNode类型，如果是，
则使用TreeNode的getTreeNode方法查找。

4. 如果next域所指向的对象不是TreeNode类型，则遍历链表后面的所元素，检查这些元素的Key是否与要查找的对象的Key相等，
如果是，则返回该元素。

5. 没有查找到，则返回null

## remove
```angularjs
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}
```

该remove方法接收一个任意类型的参数，当键相等的时候就移除相对应的元素，还有另一个remove方法，它接受两个参数，当键和值都分别相等时才移除相应的元素
```angularjs
@Override
    public boolean remove(Object key, Object value) {
        return removeNode(hash(key), key, value, true, true) != null;
}
```
两个remove方法均调用了removeNode，removeNode的具体实现如下：

```angularjs
    /**
     * Implements Map.remove and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {  //1
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&  //2
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {  //3
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //4
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)  //5
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)  //6
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

如果成功移除一个键值对，会使modCount自增1，该方法的主要步骤如下：

1. 
 
2. 

3. 1-3步为查找要移除的元素的步骤，和上面的get方法没有太大差别，这里不再赘述

4. 判断node变量是否为空，若为空则说明没有查找到要删除的元素，直接返回null，若node不为空，
则检查是否只移除Key和Value都各自相等的元素，如果是，则检查Value是否相等，相等则继续进行移除操作，
不相等则返回null，如果不是，则直接进行移除操作，不管Value是否相等。

5. 判断node是不是TreeNode类型，如果是，则调用TreeNode的removeTreeNode方法进行移除操作

6. 如果node不算TreeNode类型，则判node是否为第一个节点，如果是，则使该节点对应索引位置的元素直接指向该节点的下一个节点，
如果不是，则让node节点的前一个节点的next域指向node节点的下一个节点。


## clear
```angularjs
/**
     * Removes all of the mappings from this map.
     * The map will be empty after this call returns.
     */
    public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
}
```

移除所有的键值对，该操作会使modCount自增1，将size域设置为0，并且清除table数组上所有元素的引用，让内存得到释放。


