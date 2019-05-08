---
title: JDK1.8 HashMap源码
categories: 
- Java集合
tags: Java
---


## HashMap的数据结构
　　JDK 1.8对HashMap进行了比较大的优化，底层实现由之前的**“数组+链表”改为“数组+链表+红黑树”**。当链表节点较少时仍然是以链表存在，当链表节点较多时（大于8）会转为红黑树。
　　接下来让我们看看HashMap是如何定义的：
```java
    public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable {
```
　　在源代码中我们不难看出HashMap继承于AbstractMap，实现了Map、Cloneable、java.io.Serializable接口。
　　**AbstractMap** 是继承于Map的抽象类，它实现了Map中的大部分API。HashMap可以通过继承AbstractMap来减少重复编码。
　　实现了**Map接口**,说明HashMap中数据是以键值对的方式存储的。
　　实现了**Cloneable接口**，说明HashMap可以被克隆。
　　实现了**java.io.Serializable接口**，说明HashMap支持序列化，能通过序列化去传输。
<!-- more -->

## HashMap的源代码解析
### HashMap属性

{% fold 点击显/隐内容 %}
```java
    /**
     * The default initial capacity - MUST be a power of two.
     * HashMap默认的初始容量（1 << 4 = 16），必须是二的倍数。
     */
    //fdsafda
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * hashmap的最大存储容量为2的30次方。
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;
	/**
     * 当同一个hash值的节点数不小于8时，不再采用单链表形式存储，而是采用红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;
    /**
     * 当执行resize操作时，当桶中bin的数量少于UNTREEIFY_THRESHOLD时使用链表来代替树。默认值是6 。
     */
    static final int UNTREEIFY_THRESHOLD = 6;
    /**
     * 哈希表的最小树形化容量
     * 当哈希表中的容量大于这个值时，表中的桶才能进行树形化，否则桶内元素太多时会扩容，而不是树形化。
     * 为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
     */
    static final int MIN_TREEIFY_CAPACITY = 64;
    /**
     * HashMap的底层是个Node数组（Node<K,V>[] table），在数组的具体索引位置，如果存在多个节点，则可能是以链表或红黑树的形式存在。
     */
    transient Node<K,V>[] table;
    /**
     * 存放具体元素的集
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * The number of key-value mappings contained in this map.
     * 存放元素的个数，但这个不等于数组的长度。
     */
    transient int size;

    // 每次扩容和更改map结构的计数器
    transient int modCount;  

    // 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;

    // 填充因子
    final float loadFactor;
```
 {% endfold %}






### HashMap构造函数

{% fold 点击显/隐内容 %}
```java
	//双参构造函数，传入的值分别为初始容量和加载因子。
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
    //调用双参构造
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;//加载因子
        putMapEntries(m, false);//将m中的所有元素添加至hashmap中
    }

    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```
 {% endfold %}

### HashMap增加元素(put)
　　HashMap的put方法会去调用putVal()。
　　①判断HashMap内部的数组是否为空，如果为空会进行resize()初始化操作；
　　②其次通过(n - 1) & hash得出待插入元素在数组中的位置，如果该位置上没有元素，直接放入；
　　③如果该位置上有元素，那么还有以下几种情况
　　　　a.key相同，key的hash值也相同：这种情况直接新元素替换旧元素；
　　　　b.如果计算后的节点属于树节点，那么就插入树中；
　　　　c.如果计算后的节点所在的位置为链表，那么先遍历链表，找到相同的替换；没找到插入链
　　　　表的尾部。
　　④最后说一嘴，当key值为null的时候，该元素的key的hash值为0，(n-1)&0 = 0。所以当key为
　　null的时候，该元素存放在table[0]位置上。
接下来请看代码解析：

{% fold 点击显/隐内容 %}
```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}


 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
	//当table为空数组的时候调用resize()进行初始化.通常第一次put的时候会用到。
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
	//如果计算得出的位置上没有元素，就直接放到该位置上。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {//计算得出的位置上有元素
            Node<K,V> e; K k;
            //如果两个元素的key值相同，key的hash值也相同则替换
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果所查位置上的元素属于红黑树元素，则将新元素插入红黑树。
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {//不是红黑树那就肯定是链表了
                //遍历链表，如果找到则替换，否则插入到链表的尾部。
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果链表的长度超过8，则转成红黑树。
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //开始删除了，但红黑树的删除节点会对树进行修复的，后面会单独写一篇关于红黑树的内容。
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
 {% endfold %}

### HashMap的移除元素方法(remove)
　　HashMap的移除：方法和之前方法的思路大概类似：首先一定要确定元素的位置，其次要判断是否存储该元素，如果找到了就进行移除操作，没有则返回Null。
```java
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            //以后我们看到hash ‘与’运算首先应该想到确定位置。
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            //需要移除的元素的哈希值与存储元素的key
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                //如果确定了的元素所属于树
                if (p instanceof TreeNode)
                    //在树中找到对应节点
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    //不是树，那么就遍历链表去对比。
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
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
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


### HashMap的查询方法(get)

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    //首先判断一下table不等于空并且根据hash算法算出的位置上的元素也不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //如果算出的位置的第一个元素就是所找的，那么直接返回。
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        //如果不是，遍历红黑树或链表
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
 ```


　　这里说一下我当时的误区。我错把first当成table的第一个元素了，而且还问了很多人。最后找了很多资料重新仔细看源代码才发现这里的first是**链表的首元素**，那么一切逻辑都理通了，希望大家别犯我这样的低级错误。
　　接下来我们探讨一下高效的**除模取余运算(n-1)&hash**
　　我们都了解传统的取余运算%  如3 % 2 = 1。这个取余的过程是现将十进制数转换成二进制到内存中运算得出结果后再转成十进制。而位运算是**直接**在内存中进行，**不需要**经过这些转换.
　　但是位运算只能用于除数是2的n次方的数的求余
　　也就是说，B%C，要满足C=2^n
　　比如：
　　　　14%4 等价于 14&(2^2-1)
　　　　结果都是等于2
　　　　但是14%6  不等价于   14&6



### HashMap的扩容方法(resize)

{% fold 点击显/隐内容 %}
```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        //原hashmap的容量
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {//原hashmap内有元素
            //原HashMap的容量达到最大值
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;//直接返回原hashmap,因为无法再扩容了
            }
            //newCap = oldCap << 1,容量翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using 
            //进入该else 则证明创建hashmap
            //初始的阈值为0的话（hashmap初始化）就将默认对的参数赋予新的hashmap。ßß
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
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
 {% endfold %}














##参考
java学习--高效的除模取余运算(n-1)&hash
https://www.cnblogs.com/gne-hwz/p/10060260.html