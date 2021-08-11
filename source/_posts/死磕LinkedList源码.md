---
layout:    post
title:     死磕LinkedList源码
category:  源码解析
description: 死磕LinkedList源码
tags: JDK
date: 2021/08/11 19:42:10


---

LinkedList底层使用的是双向链表来保存元素的。继承自`AbstractSequentialList` 实现了List、Deque、Cloneable、Serializable接口。

Deque接口表明是双向链表，说明LinkedList支持双向链表的特性。

还是看下属性

```java
//长度
transient int size = 0;

//头结点
transient Node<E> first;

//尾结点
transient Node<E> last;
```

先看下Node长什么样吧

```java
//item 表示元素 next表示后面的节点 prev表示前面的节点
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

链表和数组操作不太一样，一般就是头插法和尾插法。

```java
public void addFirst(E e) {
        linkFirst(e);
        }
private void linkFirst(E e) {
//先拿到当前的第一个节点
final Node<E> f = first;
//创建一个新的结点
final Node<E> newNode = new Node<>(null, e, f);
        //将新的节点放到第一个节点的位置上
        first = newNode;
        if (f == null)
        //如果头节点为空 把新的节点赋值给最后一个节点 此时头节点 尾节点都为同一个节点
        last = newNode;
        else
        //头节点有值 那么把新的节点赋值给原来头节点的前驱节点
        f.prev = newNode;
        size++;
        modCount++;
        }

public void addLast(E e) {
        linkLast(e);
        }

//与头插法类似
        void linkLast(E e) {
final Node<E> l = last;
final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
        first = newNode;
        else
        l.next = newNode;
        size++;
        modCount++;
        }
```

假设链表此时为空，那么插入一个元素1，此时first节点就是1，该节点的前驱节点后继节点都为空，last节点也为1。然后再从头部插入一个元素2，元素2节点的后继节点为1的那个节点，然后头节点变成元素2的那个节点，元素1的前驱节点变成元素2的节点。

这里说一下双向链表和单向链表，单向链表就是从头到尾只有这种关系的链表，前一个节点只保存后一个节点的位置和自身的值，只可以从前往后遍历。双向链表是每一个节点保存一个前驱节点和一个后续节点以及自身的值，可以向前也可以向后遍历。

删除的元素的话，也是和插入一样，可以从头部删除也可以从尾部删除。

```java
public E removeFirst() {
final Node<E> f = first;
        if (f == null)
        throw new NoSuchElementException();
        return unlinkFirst(f);
        }

private E unlinkFirst(Node<E> f) {
// assert f == first && f != null;
final E element = f.item;
//拿到当前节点的下一个节点
final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        //将头节点指向下一个节点
        first = next;
        if (next == null)
        //此时链表为空了所以last要清空
        last = null;
        else
        next.prev = null;
        size--;
        modCount++;
        return element;
        }

public E removeLast() {
final Node<E> l = last;
        if (l == null)
        throw new NoSuchElementException();
        return unlinkLast(l);
        }

private E unlinkLast(Node<E> l) {
// assert l == last && l != null;
final E element = l.item;
final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
        first = null;
        else
        prev.next = null;
        size--;
        modCount++;
        return element;
        }
```

还有一种根据对象去删除元素，其实就是遍历去找这个元素，然后把这个元素的前驱节点和后继节点直接连接起来。

```java
public boolean remove(Object o) {
        if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
        if (x.item == null) {
        unlink(x);
        return true;
        }
        }
        } else {
        for (Node<E> x = first; x != null; x = x.next) {
        if (o.equals(x.item)) {
        unlink(x);
        return true;
        }
        }
        }
        return false;
        }

        E unlink(Node<E> x) {
// assert x != null;
final E element = x.item;
final Node<E> next = x.next;
final Node<E> prev = x.prev;

        //如果当前节点前驱节点为空 那么只要将头节点指向当前节点的后继节点即可
        if (prev == null) {
        first = next;
        } else {
        //将前驱节点的后继节点指向当前节点的后继节点
        prev.next = next;
        x.prev = null;
        }

        //如果当前节点后继节点为空 那么只要将尾节点指向当前节点的前驱节点即可
        if (next == null) {
        last = prev;
        } else {
        //将后继节点的前驱节点指向当前节点的前驱节点
        next.prev = prev;
        x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
        }
```

看下根据index查找Node的方法

```java
Node<E> node(int index) {
        // assert isElementIndex(index);

        //当index<(size/2)的时候 那么从前往后找
        if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
        x = x.next;
        return x;
        } else {
        //当(index>=size/2)的时候 从后往前找
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
        x = x.prev;
        return x;
        }
        }
```

其实就是为了找到一个离index比较近的方向，然后从那个方向开始遍历。

然后看下往指定位置插入元素的方法

```java
public void add(int index, E element) {
        checkPositionIndex(index);
        //当index等于size时 直接往最后插入 其实这里有点奇怪 为什么不把往头部插入也一起判断出来呢
        if (index == size)
        linkLast(element);
        else
        linkBefore(element, node(index));
        }

//其实这个思路就是新创建一个节点 然后将当前节点的前驱节点指向新创建的节点
        void linkBefore(E e, Node<E> succ) {
// assert succ != null;
//拿到当前节点的前驱节点
final Node<E> pred = succ.prev;
//创建新的节点
final Node<E> newNode = new Node<>(pred, e, succ);
        //当前节点的前驱指向新创建的节点
        succ.prev = newNode;
        if (pred == null)
        first = newNode;
        else
        pred.next = newNode;
        size++;
        modCount++;
        }
```

所以以上这些设计，导致了LinkedList的插入删除操作成本较低，因为没有扩容，但是随机访问效率不如ArrayList。所以可以根据实际场景选择合适的集合。

