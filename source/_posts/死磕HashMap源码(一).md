---
layout:    post
title:     死磕HashMap源码(一)
category:  源码解析
description: 死磕HashMap源码(一)
tags: JDK
date: 2021/08/12 9:42:10



---

> 本文有一部分思路参考自 https://segmentfault.com/a/1190000015812438

HashMap是我们平时常用到的kev-value存储的集合。首先来看类定义.

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable{}
```

继承自AbstractMap，其实就是提供一些Map的通用实现，来减少新开发一个Map时不必要的代码。

实现Map接口 Map就是K-V结构的顶层接口。像List就是集合层次中的根接口。

Cloneable、Serializable 这两个接口都是标记作用。

看下HashMap的字段定义

```java
//默认初始化容量 必须为2的幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

//最大容量 最大为2的30次幂
static final int MAXIMUM_CAPACITY = 1 << 30;

//负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

//树化阈值 这个值必须大于2小于8
static final int TREEIFY_THRESHOLD = 8;

//解构阈值 应小于TREEIFY_THRESHOLD 并且最大为6
static final int UNTREEIFY_THRESHOLD = 6;

//bin树化最小容量 就是链表的最小长度 
//应至少为 4 * TREEIFY_THRESHOLD，以避免调整大小和树化阈值之间发生冲突
static final int MIN_TREEIFY_CAPACITY = 64;

//存储数据的核心结构
transient Node<K,V>[] table;

//集合中元素个数
transient int size;

//修改次数
transient int modCount;

//要调整大小的下一个大小值（容量 * 负载因子）
int threshold;

//哈希表中的负载因子
final float loadFactor;
```

我们知道HashMap是用数组和链表构成的数据结构。HashMap会用一个指针数组table[]来分散所有的key，当一个key被加入时，会通过Hash算法算出这个数组的下标，然后就把这个K-V插入到table[]中，如果有两个不同的key算出来的下标相同的i，那么就是发生了冲突，也叫碰撞，这样会在table[i]形成一个链表，这种解决冲突的方式叫拉链法。我们知道table[]的尺寸很小，比如只有2个，如果放入10个key的话，碰撞就会很频繁，就从原来的O(1)的查找算法，变成了链表遍历，也就是O(n)。这也是拉链法去解决冲突的缺陷。所以后面使用红黑树利用红黑树的自平衡来保证查找次数的稳定。

先来看下Node的结构。

```java
static class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  V value;
  Node<K,V> next;

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
    if (o == this)
      return true;
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

从Node属性中可以看到Node是一个单向链表，每一个节点保存后继节点的引用。

那么拉链法解决冲突的时候，是怎么插入数据的呢？在Java8之前是使用头插法，就是新来的值取代原有的值，原有的值会被推到链表中去，因为作者认为后来的值被查找的可能性更大一点，想提升查找效率。但是Java8之后，又改成了尾插法。

敲黑板！！这里是重点。为什么后来改成尾插法。

先来看下数据插入的源码。

```java
public V put(K key, V value) {
  //这里看到拿到hash码然后进行putVal
  return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
  Node<K,V>[] tab; Node<K,V> p; int n, i;
  //先看当前table是不是空的 空的先进行扩容
  if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;
  //判断当前元素在不在hash表中 如果不在 那么直接创建新的节点
  if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
  else {
    //发生碰撞
    Node<K,V> e; K k;
    //hash码相等 && key相等 这里也是为什么重写hashCode 还要重写equals
    if (p.hash == hash &&
        ((k = p.key) == key || (key != null && key.equals(k))))
      e = p;
    else if (p instanceof TreeNode)
      //当前节点已经是树节点了 那么直接使用树节点的插入方法
      e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
    else {
      //此时为链表
      for (int binCount = 0; ; ++binCount) {
        if ((e = p.next) == null) {
          //找到最后一个节点然后将新创建的节点插入 尾插法
          p.next = newNode(hash, key, value, null);
          //当链表长度大于树化阈值 树化
          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
          break;
        }
        //遍历链表中的key是否存在重复
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          break;
        p = e;
      }
    }
    //e不等于空 就是key可以认为是同一个key
    if (e != null) { // existing mapping for key
      V oldValue = e.value;
      if (!onlyIfAbsent || oldValue == null)
        //更新value
        e.value = value;
      //回调通知
      afterNodeAccess(e);
      return oldValue;
    }
  }
  ++modCount;
  if (++size > threshold)
    //扩容
    resize();
  //回调通知
  afterNodeInsertion(evict);
  return null;
}
```

这里说一下为什么是用(n - 1) & hash。这个是一个取余操作。取余操作中如果除数是2的幂则等价于与其除数减一的与(&)操作，hash%length==hash&(length-1)，前提是长度2的n次方，采用二进制位操作能提高运算效率。这也解释了为什么容量必须是2的幂了。

看下扩容代码

```java
final Node<K,V>[] resize() {
  Node<K,V>[] oldTab = table;
  int oldCap = (oldTab == null) ? 0 : oldTab.length;
  //以前的扩容阈值
  int oldThr = threshold;
  int newCap, newThr = 0;
  if (oldCap > 0) {
    //容量已经最大 那么直接修改threshold返回
    if (oldCap >= MAXIMUM_CAPACITY) {
      threshold = Integer.MAX_VALUE;
      return oldTab;
    }
    //旧容量小于最大值 threshold直接翻倍 容量也翻倍
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY)
      newThr = oldThr << 1; // double threshold
  }
  else if (oldThr > 0) // initial capacity was placed in threshold
    newCap = oldThr;
  else {               // zero initial threshold signifies using defaults
    newCap = DEFAULT_INITIAL_CAPACITY;
    //新的容量上限 = 负载因子 * 容量
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
  }
  //其实这里比较绕 就是创建的时候指定初始化容量如1 然后第一个数据进来 进入resize中
  if (newThr == 0) {
    //2*0.75 = 1.5 这里newCap是因为上面进行了容量翻倍
    float ft = (float)newCap * loadFactor;
    //newThr = 1
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
              (int)ft : Integer.MAX_VALUE);
  }
  threshold = newThr;
  @SuppressWarnings({"rawtypes","unchecked"})
  Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
  table = newTab;
  if (oldTab != null) {
    for (int j = 0; j < oldCap; ++j) {
      //遍历旧的table
      Node<K,V> e;
      if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        if (e.next == null)
          //如果当前节点不为空并且后继节点为空 直接在newTab找到位置赋值
          newTab[e.hash & (newCap - 1)] = e;
        else if (e instanceof TreeNode)
          //如果是树的结构 进行树的处理
          ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
        else { // preserve order
          //如果是链表 需要保持原来的顺序
          //这里根据字段名猜测应该是2个链表 一个lo队列 一个hi队列
          Node<K,V> loHead = null, loTail = null;
          Node<K,V> hiHead = null, hiTail = null;
          Node<K,V> next;
          do {
            next = e.next;
            //这里根据e.hash & oldCap的结果来判断节点属于lo还是hi
            if ((e.hash & oldCap) == 0) {
              //尾插法
              if (loTail == null)
                loHead = e;
              else
                loTail.next = e;
              loTail = e;
            }
            else {
              //e.hash & oldCap 结果只会为0或者为1
              if (hiTail == null)
                hiHead = e;
              else
                hiTail.next = e;
              hiTail = e;
            }
          } while ((e = next) != null);
          //如果loTail非空 那么将lo链表放到newTab[j]的位置上
          if (loTail != null) {
            loTail.next = null;
            newTab[j] = loHead;
          }
          //如果hiTail非空 那么将hi链表放到newTab[j+oldCap]的位置上
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

首先来说下，HashMap扩容是2倍扩容的。所以也就意味着原来链表里的key有两个去处，要么是newTab[j] 要么是newTab[j+oldCap]。这个分链表的操作 其实有点像我们站队，老师让我们一二报数，然后按照喊的数字去分组一样。`e.hash & oldCap` 这个操作就是决定这个节点到底喊的是0还是1的关键。

有三点明确下：

-  oldCap一定是2的整数次幂，这里假设是2^m
- newCap是oldCap的两倍，则newCap为2^(m+1)
- hash对数组大小取模(n-1)&hash其实就是取hash的低m位

假设oldCap = 16 即2^4

16-1 = 15 二进制表示为 `0000 0000 0000 0000 0000 0000 0000 1111` 除了低四位，高位都是0。所以(16-1)&hash 也就可以认为是低四位&hashcode，假设它为abcd。

当oldCap扩大两倍后，新的index就应该是(32-1)&hash，其实就是取hash值的低五位。此时也就2中情况第五位不是0就是1。也就是0abcd、或者1abcd。其中0abcd和原来的值一样，而1abcd = 0abcd + 1 0000 也就是差了一个oldCap。所以同一个key要么和原来的index一致，要么差了一个oldCap。

如何拿到第五位的值呢？

hash & 0000 0000 0000 0000 0000 0000 0001 0000 也就等效于 hash & olcCap

所以可以得出（e.hash & oldCap) == 0 该节点在新表中的下标位置于旧表一致，（e.hash & oldCap) == 1该节点在新表的下标位置为 j + oldCap

HashMap中，决定key位置的就是hash函数，看下哈希函数如何实现的

```java
static final int hash(Object key) {
  int h;
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

这里看到HashMap对hashCode进行了高16位和低16位进行了异或，这充分利用了高半位和低半位的信息，对低位进行扰动，目的就是使该hashCode映射成数组下标是可以更分散。以初始长度为例，16-1=15，与某散列值“与”操作如下

```
  	10100101 11000100 00100101
&		00000000 00000000 00001111
----------------------------------
		00000000 00000000 00000101       //高位全部归零，只保留末四位
```

如果散列值算的不够分散，只取最后几位的话，分布上成等差数列的话，碰撞就会很严重。这时候扰动函数的价值就出来了。

![](/blog/images/hashmap/hashcode.png)

右移16位，正好是32bit的一半，自己的高半区和低半区做异或，就是为了混合原始哈希吗的高位和低位，以此来加大随机性。而且混合后的低位参杂了高位的部分特征，这样高位的信息也被变相保留下来。

总结一下HashMap插入数据流程：

1.对key重新计算一个扰动过的hash值，(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);

2.看table是否为空或者长度为0，如果是则进行扩容 if ((tab = table) == null || (n = tab.length) == 0)  n = (tab = resize()).length;

3.根据hash值计算index的位置，如果该index没有存放数据，则直接插入，如果有数据，那么需要判断key是否相等以及equals方法是否相等，如果相等则覆盖，不相等时，先判断是不是树节点，树节点执行树节点的插入逻辑，否则就是链表，使用拉链法解决冲突，拉链法解决冲突的时候也会检查链表长度是否大于TREEIFY_THRESHOLD阈值，进而执行判断是否需要树化的逻辑。

4.插入数据完毕，检查当前插入数据是否大于设定容量，进而进行扩容操作。