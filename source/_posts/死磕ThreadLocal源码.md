---
layout:    post
title:     死磕ThreadLocal源码
category:  源码解析
description: 死磕ThreadLocal源码
tags: JDK
date: 2021/08/13 21:52:10
---

`ThreadLocal` 是我们常用的和线程绑定的线程安全的对象，别的线程是无法访问的，所以该对象不存在线程安全问题。今天就来看下源码，看下ThreadLocal是怎么做的吧。

先看下用法

```java
private static ThreadLocal threadLocal = new ThreadLocal();
threadLocal.set(new SimpleDateFormat("yyyy-MM-dd ss:hh:ss"));
```

##### 插入

这里直接看set()方法做了点啥

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

看到这里可以知道ThreadLocal的实现其实就是通过Thread中ThreadLocalMap类型的threadLocals去实现的。可能比较绕，我来描述一下。我们通过`ThreadLocal`对象调用set方法，其实是往`Thread类`中的`threadLocals`这个`ThreadLocalMap`中存值，然后这个`ThreadLocalMap`中`key`就是`ThreadLocal`对象，`value`就是我们`set`的`value`。

这里看到set()根据map是否为空执行了两种逻辑，一种是直接创建一个ThreadLocalMap，一种是往已有的ThreadLocalMap中插入数据。

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //创建保存对象的table
    table = new Entry[INITIAL_CAPACITY];
    //寻找这个threadLocal应该放的index
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    //创建Entry对象放入table中
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}

//这个Entry一会儿要用，我先拿过来 可以看到这个不是一个链表
//因为ThreadLocalMap的key就是ThreadLocal对象 如果同一个threadLocal，set多次后面的值会覆盖 因为key相同
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}

private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    //如果tab[i]没有值，不进入循环 否则一直循环 直到找到一个tab[i] == null为止
    //就是如果算出来的i对应的tab[i]被占了，就会往后拿桶
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
		
        //key相等 直接覆盖
        if (k == key) {
            e.value = value;
            return;
        }

        //如果key为nul，说明该key被GC了，用当前值替换老值
        //注意，这里不能直接将value替换成e.value是因为当前的key已经是null了，无法确定key是否和当前的key一样
        //仅仅是计算出来的index一样，需要遍历整个table去找对应的key 再根据实际情况进行赋值
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    //做一次过期清理，如果没有元素被清理并且元素值超过扩容阈值 进行rehash扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    //往前检查 记录下最后一个不为空并且key为null的元素的位置 以便后续的删除
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    //往后检查 元素不为空并且key相等 就用value替换 直到元素为空
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        //找到需要被替换的过期元素
        if (k == key) {
            //替换value
            e.value = value;

            //交换 table[i]和table[staleSolt]
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
            //重置slotToExpunge为i
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            //执行清理逻辑
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        //如果前面没有找到过期的元素 slotToExpunge的位置应该从i开始
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 如果没有找到匹配的元素 则直接替换把新的值赋值给staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    //如果存在需要被清理的元素 执行清理
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

清理的逻辑如下

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```



这里看到getMap()这个方法又从Thread中拿了个`threadLocals`返回了，这个threadLocals在Thread类中定义如下：

```java
ThreadLocal.ThreadLocalMap threadLocals = null;
```

那这个ThreadLocal.ThreadLocalMap又是个啥呢？看名字的话，知道它是个Map一样的结构。不过它的这些Map相关的操作都是自己实现的，总体逻辑其实和HashMap是类似的。

##### 获取

获取通过get()方法。

```java
public T get() {
    Thread t = Thread.currentThread();
    //拿到当前线程的threadLocals
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //如果上面没有找到ThreadLocal对应的值，尝试调用setInitialValue来取值，因为可能重写了initialValue方法来获取值 
    //ThreadLocal threadLocal = ThreadLocal.withInitial(()->new SimpleDateFormat("yyyy-MM-dd ss:hh:ss")); 通过这种方式创建
    return setInitialValue();
}

private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}

//因为我们遇到hash冲突的时候不是拉链法去解决的，而是往后找了一个空的位置放进去，所以查询的时候也需要特殊处理一下
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e); //这里就是特殊处理的地方
}

//这个就是如果没有找到就去下一slot去拿 直到拿到元素为止
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}

//这个方法主要是
//删除staleSlot的值
//从staleSlot位置开始，直到下一个table[i]为null的元素之间，删除table[i].key为null的元素，同时调整该区间内的元素位置到合适的位置
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    //删除staleSlot位置的元素
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    //从staleSlot开始查找需要删除或者更新位置的元素
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

##### 删除

与Map的remove(key)思想一致。

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

#### 总结一下

ThreadLocalMap的key是WeekReference的，这就决定了，在GC时，只要没有任何强引用指向Map中的key，那么该key就会在下一次GC时被回收，从而key.get()就会返回null，而在set、get等方法时，会对key.get()为null的KV键值对进行清理操作，从而释放内存，避免内存泄露。但是和WeekHashMap一样，即使key被GC掉了，变成null，但是如果从此之后不再对map做任何操作，那么这些空key对应的value所占用的内存也是不可能被GC的，这种情况下，有可能发生内存泄漏。

ThreadLocalMap对hash冲突的处理不是通过拉链法来解决的，而是从当前计算的index往后寻找为空的slot。

虽然说ThreadLocalMap中的key是弱引用，但是一旦外面还有其他强引用指向该key，意味着该key永远都不可能会被gc掉，那么WeakReferece的能力就基本失去了作用，这就导致内存无法回收的后果，所以每次使用完，确定不再需要该key，就需要调用ThreadLocal的remove方法或者set(null)来帮助GC回收相关内存。

#### 比较容易踩的坑

1.使用线程池处理事务时，如果同一线程上一次处理过的ThreadLocal对象，在用完后没有及时清理，很有可能导致该线程在处理下一个事务时，由于没有重新设置该ThreadLocal对象，而导致处理下一个事务的时候，读取到了上一次处理过程中ThreadLocal中的值。

2.内存泄露问题，因为一般情况下，ThreadLocal对象可能是静态对象，意味着在Thread的ThreadLocalMap中该ThreadLocal的弱引用失效（因为静态对象的引用是强引用），从而无法达到key被GC掉，并且在后续操作Thread的ThreadLocalMap时，无法自动将不再需要的对象清除掉而造成内存泄漏的问题。



