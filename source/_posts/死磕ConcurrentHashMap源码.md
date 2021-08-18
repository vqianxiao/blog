---
layout:    post
title:     死磕ConcurrentHashMap源码
category:  源码解析
description: 死磕ConcurrentHashMap源码
tags: JDK
date: 2021/08/16 20:10:10
---

由于HashMap不是线程安全的，替代方案有三种：

- 使用Collections.synchronizedMap(Map)创建线程安全的map集合
- HashTable
- ConcurrentHashMap

不过出于线程并发度的原因，一般都会选择ConcurrentHashMap。先来看下第一种是怎么实现的。

```java
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m) {
    return new SynchronizedMap<>(m);
}

private static class SynchronizedMap<K,V>
    implements Map<K,V>, Serializable {
    private static final long serialVersionUID = 1978198479659022715L;

    private final Map<K,V> m;     // Backing Map
    final Object      mutex;        // Object on which to synchronize

    SynchronizedMap(Map<K,V> m) {
        this.m = Objects.requireNonNull(m);
        mutex = this;
    }

    SynchronizedMap(Map<K,V> m, Object mutex) {
        this.m = m;
        this.mutex = mutex;
    }
}

public V get(Object key) {
    synchronized (mutex) {return m.get(key);}
}

public V put(K key, V value) {
    synchronized (mutex) {return m.put(key, value);}
}

public V remove(Object key) {
    synchronized (mutex) {return m.remove(key);}
}
```

可以看到SynchronizedMap内部维护了一个Map对象，还有一个互斥锁mutex。然后看我们常用的get()、put()、remove()方法，其实就是在调用实际执行方法的时候加锁。

然后看下HashTable，为了节省篇幅 我就粘一个方法

```java
 public synchronized V put(K key, V value) {
     // Make sure the value is not null
     if (value == null) {
         throw new NullPointerException();
     }

     // Makes sure the key is not already in the hashtable.
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
```

其实和上面的实现类似，都是直接上锁，所以效率比较低。

如果仔细看可能就发现，这个HashTable和HashMap存在一些区别，这个HashTable不允许键值对为空，就是key和value都不能为空。因为value == null 会直接抛NPE，然后key去拿hashCode的时候也会抛NPE。

然后该说我们的重头戏，ConcurrentHashMap了。

ConcurrentHashMap是一个`线程安全`、`高吞吐`、`低时延`的数据结构。

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    private static final long serialVersionUID = 7249069246763182397L;

    //最大表容量
    private static final int MAXIMUM_CAPACITY = 1 << 30;

    //默认初始容量
    private static final int DEFAULT_CAPACITY = 16;

    //可能的最大数组大小
    static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    //默认并发级别
    private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

    //负载因子
    private static final float LOAD_FACTOR = 0.75f;

    //树化阈值
    static final int TREEIFY_THRESHOLD = 8;

    //树退化阈值
    static final int UNTREEIFY_THRESHOLD = 6;

    //树化的最小表容量
    static final int MIN_TREEIFY_CAPACITY = 64;

    private static final int MIN_TRANSFER_STRIDE = 16;

    private static int RESIZE_STAMP_BITS = 16;

    //帮助resize最大线程数
    private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;

    private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

    static final int MOVED     = -1; // hash for forwarding nodes
    static final int TREEBIN   = -2; // hash for roots of trees
    static final int RESERVED  = -3; // hash for transient reservations
    static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash

    //cpu个数
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    /** For serialization compatibility. */
    private static final ObjectStreamField[] serialPersistentFields = {
        new ObjectStreamField("segments", Segment[].class),
        new ObjectStreamField("segmentMask", Integer.TYPE),
        new ObjectStreamField("segmentShift", Integer.TYPE)
    };
    
    
    transient volatile Node<K,V>[] table;

    
    private transient volatile Node<K,V>[] nextTable;

    
    private transient volatile long baseCount;

    //表初始化和大小控制 负数表示初始化或者调整大小 -1表示初始化 table为null保存创建时使用的初始表大小，或默认为 0
    //初始化后，保存下一个要调整表格大小的元素计数值。
    private transient volatile int sizeCtl;

    
    private transient volatile int transferIndex;

    
    private transient volatile int cellsBusy;

    
    private transient volatile CounterCell[] counterCells;

    // views
    private transient KeySetView<K,V> keySet;
    private transient ValuesView<K,V> values;
    private transient EntrySetView<K,V> entrySet;

    // Unsafe mechanics
    private static final sun.misc.Unsafe U;
    private static final long SIZECTL;
    private static final long TRANSFERINDEX;
    private static final long BASECOUNT;
    private static final long CELLSBUSY;
    private static final long CELLVALUE;
    private static final long ABASE;
    private static final int ASHIFT;

    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentHashMap.class;
            SIZECTL = U.objectFieldOffset
                (k.getDeclaredField("sizeCtl"));
            TRANSFERINDEX = U.objectFieldOffset
                (k.getDeclaredField("transferIndex"));
            BASECOUNT = U.objectFieldOffset
                (k.getDeclaredField("baseCount"));
            CELLSBUSY = U.objectFieldOffset
                (k.getDeclaredField("cellsBusy"));
            Class<?> ck = CounterCell.class;
            CELLVALUE = U.objectFieldOffset
                (ck.getDeclaredField("value"));
            Class<?> ak = Node[].class;
            ABASE = U.arrayBaseOffset(ak);
            int scale = U.arrayIndexScale(ak);
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

#### 插入

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    //这里重新计算了一下hash  (h ^ (h >>> 16)) & HASH_BITS
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        //如果table为空 那么初始化table
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //获取当前元素
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //slot空的时候添加没有锁 
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //表示当前slot上的元素已经迁移到新的table上，但是还未完成扩容过程，调用帮助扩容方法，其实最终调用的是transfer(tab,nextTab)
        //nextTab就是扩容后的新表
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //这个锁的粒度是Node
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                //key相同 覆盖
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            //否则拉链法解决冲突
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        //putTreeVal 这个方法是查找或者添加节点 如果添加返回null 如果找到key相同则直接返回TreeNode
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                              value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    //树化
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

看下创建table的代码

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        //compareAndSwapInt 第一个参数 要更新的对象
        //第二个参数 实例变量的内存地址偏移量
        //第三个参数 预期的旧值
        //第四个参数 要更新的值
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    //相当于容量减去1/4 也就是sc为容量*0.75
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

slot空的时候添加node，这个时候没有锁，这是用CAS去添加。

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

总结一下ConcurrentHashMap插入操作步骤：

1.key-value空值判断，如果为空直接抛NPE

2.对key的hashCode做`spread`运算，算法就是用hashcode的高16位与低16位做异或运算，并与`0x7fffffff`进行与运算，目的就是为了让hash值更分散

3.执行插入操作

- 如果table为空，则执行初始化工作
- 如果key对应的位置为null，执行`CAS`操作，将KV创建成Node插入，成功即返回，失败则自旋保证成功
- 如果key对应的位置有值，并且hash值为MOVED，表示当前slot上的所有元素已经迁移到新的table上，但是还未完成扩容，则进行尝试帮助扩容，也就是调用`helpTransfer(tab,f)`方法
- 对当前slot加锁
    - 当hash值大于等于0，就是链表节点，如果存在相等的key，则覆盖value。否则往该节点上插入新的节点
    - 当`Node`为`TreeBin`类型，则调用`putTreeVal()`方法去插入元素。还是存在相等的key，覆盖value并返回，否则插入新的元素，返回null。
- 对slot的长度进行检查，如果长度大于8，则调用`treeifyBin()`方法去检查是否需要树化，如果slot长度大于8并且table的长度大于64，则会进行树化。否则进行扩容
- 调用`addCount()`更新size

这里其实我看完代码其实一直有个疑惑，这个hash值计算的结果明明都是正数，怎么到了树节点就成了负数呢？

在转移Node的时候，会有一个`ForwardingNode`，这个类的构造函数里hash值写死是MOVED也就是-1。所以有了上面的如果hash值为MOVED的时候，去帮助transfer了。

在转成树的时候，会有一个`TreeBin`，这个类的构造函数hash也是写死为TREEBIN也就是-2。所以有了树化的时候，hash值为-2了。

但是这里的`ForwardingNode`是在转移的时候放在头部的节点，是一个空节点，来表示当前节点已经在进行扩容转移了。

而`TreeBin`则表示当前节点是红黑树，在构建好红黑树后，将`first`指向链表第一个元素（用来维护原来链表的插入顺序，树退化的时候可以恢复成链表原来的样子），将`root`指向红黑树的根节点。

#### 查询

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    //计算hash值
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        //从链表中循环查找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

可以看到get方法比较简单，直接通过计算hash值然后去找到对应的slot，然后去遍历该位置的所有节点，直到找到相等的那个key。

#### 总结

扩容的条件：

- 向map中添加元素的时候，某一节点数目超过8个，并且table长度小于64时，才会触发扩容操作
- 当数组中元素达到了sizeCtl的数量的时候，则会调用transfer方法来扩容

扩容的时候，是否还可以进行读写操作呢？

当数组扩容的时候，如果当前节点还没被处理（还没有设置为fwd节点），那就可以进行赋值操作。如果该节点已被处理，则当前线程也加入到扩容操作中去。

多线程的时候，如何进行同步处理的呢？

在ConcurrentHashMap中，同步处理主要通过Synchronized和unsafe两种方式来完成。

- 在获取sizeCtl、某个index的Node的时候，都是通过unsafe方法，来达到并发安全的目的
- 当需要操作数组中的某个位置的节点时，会通过synchronized同步机制，锁定该位置的节点
- 在数组扩容的时候，通过处理的步长和fwd节点来达到并发安全的目的，通过设置hash值为MOVED
- 当把某个位置的节点赋值到新的table的时候，也通过synchronized的同步机制来保证线程安全

