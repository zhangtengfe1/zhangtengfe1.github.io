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
    

    public synchronized void putAll(Map<? extends K, ? extends V> t) {
        for (Map.Entry<? extends K, ? extends V> e : t.entrySet())
            put(e.getKey(), e.getValue());
    }
```


```java
    public synchronized V remove(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                modCount++;
                if (prev != null) {
                    prev.next = e.next;
                } else {
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

