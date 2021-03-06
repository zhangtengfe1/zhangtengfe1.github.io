---
title: JDK1.8 HashTable源码
categories: 
- Java集合
tags: Java
---

## HashTable介绍
```java
	public class Hashtable<K,V>
	    extends Dictionary<K,V>
	    implements Map<K,V>, Cloneable, java.io.Serializable 
```
　　HashTable是一个散列表，它存储的内容是（key-value）键值对。
　　HashTable继承于Directionary,Directionary也是键值对的接口，并且HashTable也实现了Map接口，存储键值对毋庸置疑。Cloneable接口使它可以被克隆，Serializable使它可以被序列化。HashTable是**无序的**。

<!-- more -->

## 源码分析
### 构造函数

```java
	//由此可见，HashTable的默认容量为11，默认加载因子为0.75。
	public Hashtable() {
        this(11, 0.75f);
    }

	public Hashtable(int initialCapacity) {
        this(initialCapacity, 0.75f);
    }

    //双参的构造，看过之前hashmap的就都懂了，无非是各种判断然后赋值。
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

    public Hashtable(Map<? extends K, ? extends V> t) {
        this(Math.max(2*t.size(), 11), 0.75f);
        putAll(t);
    }
```

### HashTable的API

```java
	public synchronized int size()

	public synchronized boolean isEmpty()

	public synchronized Enumeration<K> keys()

	public synchronized boolean contains(Object value)

	public synchronized boolean containsKey(Object key)

	public synchronized V get(Object key)

	public synchronized V put(K key, V value)

	public synchronized V remove(Object key) 

	public synchronized void putAll(Map<? extends K, ? extends V> t)

	public synchronized void clear()

	public synchronized Object clone()

	public synchronized String toString()

	public synchronized boolean equals(Object o)

	public synchronized void replaceAll(BiFunction<? super K, ? super V, ? extends V> function)

	public synchronized boolean remove(Object key, Object value)

	public synchronized boolean replace(K key, V oldValue, V newValue)

```
　　列举的都是我们在生活中常见的hashtable  api。我们会发现他们都由**synchronized**修饰，所以是线程安全的。hashtable和hashmap结构上几乎相似（hashtable没有红黑树），所以经常被拿来和hashmap作比较。

　　还是老规矩，增删查走起！！！
### put()
　　① 先获取synchroized锁
　　② 对添加的数据的值进行判断，如果为null会抛出异常
　　③ 计算key在数组中的位置
　　④ 遍历链表，如果发现已经存在相同的key，那么就替换并返回旧值；没有则添加到链表的头部。
　　⑤ 对hashtable是否需要扩容进行判断，如要则扩容。
```java

	public synchronized V put(K key, V value) {
        // Make sure the value is not null
        //确保value值不为空，这就是hashtable是不能存储空值的原因。
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        //如果添加的key已经存在，那么就将新值替换掉旧值并返回旧值。
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
    private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry<?,?> tab[] = table;
        //如果添加的过程中table的元素个数大于等于阈值（容量*加载因子），扩容。
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        //注意，此处为 hashtable将新添加的元素放在链表的首位置。
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
    

    public synchronized void putAll(Map<? extends K, ? extends V> t) {
        for (Map.Entry<? extends K, ? extends V> e : t.entrySet())
            put(e.getKey(), e.getValue());
    }
```

### get()
　　① 先获取synchroized锁
　　② 通过计算得到元素的位置
　　③ 遍历链表，如果找到则返回value值，否则返回null。

```java
    public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
```

### remove()
　　首先会确定将要删除的元素在数组(table)中的位置，然后会遍历链表删除元素。删除的大致描述是将上一个元素的指针指向当前元素的下一个节点。删除成功后会返回所删除节点的value值，如果hashtable内不存在所删除元素，返回null。
```java
    public synchronized V remove(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;//求出所删除元素在数组中的位置
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                modCount++;
                if (prev != null) {//遍历链表
                    prev.next = e.next;
                } else {
                    //删除掉链表的头节点
                    tab[index] = e.next;
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

### rehash()
```java
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

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
```

