---
layout:    post
title:     死磕ArrayList源码
category:  源码解析
description: 死磕ArrayList源码
tags: JDK
date: 2021/08/11 17:11:10
---
ArrayList是List接口的可变数组的实现。实现了所有可选列表的操作。继承自AbstractList，实现了List接口、RandomAccess接口、Cloneable接口、Serializable接口。
实现的这些接口中，RandomAccess接口和Cloneable接口还有Serializable接口都是表示ArrayList拥有随机访问、克隆、序列化的能力。

Iterable是一个遍历接口，提供了获取迭代器的方法，集合可以根据自己的需要实现自己的迭代器

每一个ArrayList都有一个容量，该容量就是存储元素的数组大小。它总是大于等于列表的大小的，随着向ArrayList中不断添加元素，其容量也会自动增长，自动增长会带来数组的拷贝。因此如果提前知道数据量大小，在构造函数提前把容量设置好，可以减少因为扩容带来的数据拷贝的不必要的性能损耗。

ArrayList的属性

```java
		/**
     * Default initial capacity.
     */
		//默认容量大小
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
		//空集合的空实例
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
		//默认大小的空实例 和EMPTY_ELEMENTDATA 不同是为了知道添加第一个元素的时候需要膨胀多少
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
		//就是集合中的数组元素
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
		//集合中元素的个数
    private int size;
```

相关注释写在代码上了。然后来看add()方法。

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
}
```

这里面后面2行基本上不用讲了，主要是ensureCapacityInternal()这个方法干了点啥呢？

```java
private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

//计算容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
}

//检查容量是否足够 不够的话进行扩容
private void ensureExplicitCapacity(int minCapacity) {
  //这个modCount要注意下
        modCount++;
				
  			//当前数据量大于数组的容量了 进行扩容
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
}

//扩容
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
  			//新容量 = 旧容量 + （旧的容量/2）新的容量就是旧的容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
  			//新容量小于需要的容量 需要的容量覆盖新容量
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
  			//新的容量比最大容量大 使用最大容量
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
  			//进行数组拷贝 把旧的数据拷贝到新的数组上
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
}
```

这里有个modCount这里要讲下，modCount就是修改次数，在AbstractList中有这么一段判断

```java
final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
 }
```

就是判断modCount和expectedModCount是否相等的，如果不相等，那么抛出`ConcurrentModificationException` 异常。主要是为了防止并发操作，遍历的时候进行数据增删，可能会导致某个元素被重复遍历或遍历不到，是一种fast-fail的解决方式。

看下ArrayList的Remove()方法

```java
public E remove(int index) {
  			//index合法性检查 不能大于size
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
          //数组拷贝 将该元素后面的部分和前面的部分拼接起来 最后面多了一个空的元素
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }


```

