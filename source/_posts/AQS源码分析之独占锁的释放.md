---
layout:    post
title:     AQS源码分析之独占锁的释放
category:  JUC
description: AQS源码分析之独占锁的释放
tags:
- JDK
- JUC
  date: 2021/08/19 00:28:10

---

上一篇讲了ReentrantLock的NonfairSync的获取锁的操作，这篇主要来看一下释放锁的操作。这里要提一下，Java的内置锁在退出临界区之后会自动释放，但是ReentrantLock这样的显式锁是需要手动释放的，所以在加锁的时候一定要记得在finally语句块中释放锁！！！

#### 释放锁

由于公平锁和非公平锁的释放是一样的，所以unlock的逻辑并没有放在FairSync或NonfairSync中，而是直接定义在ReentrantLock类中：

```java
    public void unlock() {
        sync.release(1);
    }
```

release方法定义在AQS类中，描述了释放锁的流程

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

相比于获取锁，释放锁的过程简单很多，只涉及两个函数调用。

- tryRelease(arg)

  该方法由继承AQS的子类实现，为释放锁的具体逻辑

- unparkSuccessor(h)

  唤醒后继线程

这里要注意一下，只要tryRelease返回true，那么后续操作不会影响最终release返回true的结果。

if (h != null && h.waitStatus != 0) 这个条件h!=null 就是头节点不为空很好理解，那么头节点的waitStatus !=0是什么意思。在获取锁的时候，如果没有获取成功然后当前线程入队成功后，会调用shouldParkAfterFailedAcquire函数，这个函数会将前驱节点的waitStatus设为Node.SIGNAL，但是这个值也不是0呀。我们在新建一个节点的时候，在addWaiter函数中，当我们将一个新的节点添加进队列或者初始化空队列的时候，都会新建节点，而新建节点的waitStauts在没有赋值的情况下都会初始化为0。

所以当一个head节点的waitStatus为0说明这个head节点后面没有在挂起等待中的后继节点了（如果有的话，就会被后面的节点修改waitStatus为Node.SIGNAL了），自然也不执行UnparkSuccessor了。

##### tryRelease

tryRelease是由ReentrantLock的静态类Sync实现。这里要记住，能执行释放锁的线程，一定是已经拿到锁的线程。因为是已经拿到锁的线程去释放锁，所以不会有任何竞争，也没有任何CAS操作。

```java
    protected final boolean tryRelease(int releases) {
        //先将持有锁的线程数减1 unlock时传入的参数为1
        //这里的操作主要是针对可重入锁的情况 c可能是大于1的
        int c = getState() - releases;
        //释放锁的线程和持有锁的线程必须一致
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //没有线程持有锁了 所以锁可以直接释放了 否则只是减少持有锁的线程数
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
```

##### unparkSuccessor

锁成功释放之后，就需要唤醒后续节点了。这个方法定义在AQS中。

```java
 private void unparkSuccessor(Node node) {
        
        int ws = node.waitStatus;
     	//如果head节点的ws小于0直接赋值成0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        //通常情况下，要唤醒的节点是自己的后继节点
     	//如果后继节点存在并且在等待锁，那么直接唤醒它
     	//但是有可能存在后继节点取消等待锁 的情况
        //这个时候从尾节点开始向前找，直到找到最靠前的ws<=0的节点 然后唤醒它
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

当前节点的前驱节点的waitStatus这个属性是用来决定是否要挂起当前线程的，并且如果一个线程被挂起，它的前驱节点的waitStatus必然是Node.SIGNAL。Node.CANCELLED值为1，所以只要不是取消，就会被唤醒。

这里其实有一个问题，为什么要从尾节点开始遍历，而不是从头节点开始遍历呢，如果从头节点开始遍历，似乎遍历走过的路径更短。

这里从后往前遍历是有个条件的：

```java
if (s == null || s.waitStatus > 0)
```

也就是后继节点不存在，或者后继节点取消了排队，这一条件大多数情况下是不满足的。因为虽然后继节点取消排队很正常，但是节点在挂起前，都会给自己找一个waitStatus状态为SIGNAL的前驱节点，而跳过那些已经取消掉的节点。

所以从后往前找的目的是为了照顾刚加入队列的节点，这就涉及到上一篇介绍的“尾分叉”了：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode); //将当前线程包装成Node
    Node pred = tail;
    // 如果队列不为空, 则用CAS方式将当前节点设为尾节点
    if (pred != null) {
        node.prev = pred; //step 1, 设置前驱节点
        if (compareAndSetTail(pred, node)) { // step2, 将当前节点设置成新的尾节点
            pred.next = node; // step 3, 将前驱节点的next属性指向自己
            return node;
        }
    }
    enq(node); 
    return node;
}
```

仔细看上面的代码，可以发现节点入队不是一个原子操作，虽然用了compareAndSetTail操作保证了当前节点被设置成尾节点，但是只能保证，第一步和第二步是执行完成的，有可能在第三步还没被执行到的时候，unparkSuccessor方法就开始执行了，此时pred.next还没有被设置成node，所以从前往后遍历是遍历不到的，但是因为尾节点已经设置完成，node.prev = pred操作也被执行过了，所以可以根据尾节点去往前遍历。

总结一下就是，我们处于多线程条件下，如果一个节点的next属性为null，并不能保证它就是尾节点（可能因为新加的尾节点还没执行pred.next = node）,但是一个节点能入队，那么它的prev属性一定是有值的，所以反向查找一定是最精确的。

在调用LockSupport.unpark(s.thread)之后，会发生什么呢？

当然是回到最初的起点啦，从哪里被挂起来就从哪里唤醒（继续执行）：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); //在这里被挂起了, 唤醒之后就能继续往下执行了
    return Thread.interrupted();
}
```

线程被唤醒后，它将调用Thread.interrupted()并返回，Thread.interrupted()这个方法将返回当前线程的的中断状态，并清除它。然后再回到parkAndCheckInterrupt被调用的地方

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            
            // 在这里！！！在这里！！！
            // 在这里！！！在这里！！！
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

可以看到如果Thread.interrupted()返回true，那么parkAndCheckInterrupt()也会返回true，interrupted将被修改成true，反之为false。

再接下来回到for(;;)循环中，进行新一轮的抢锁。

假设这次抢到了，将从return interrupted返回，返回到acquireQueued的调用处，就是：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

如果acquireQueued返回true，将执行selfInterrupt()

```java
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

这段代码的作用就是中断当前线程。绕了这么一大圈，最后还是中断了当前线程，干啥呢这是？

这一切的原因在于，我们不知道线程唤醒的原因。

当我们从LockSupport.park(this)处被唤醒，我们并不知道是什么原因被唤醒，可能是别的线程释放锁，调用的LockSupport.unpark(s.thread)，也有可能是因为当前线程在等待中被中断了，因此我们通过Thread.interrupted()检查了当前线程的中断标志，并把它记录下来，在最后返回acquire()方法后，**如果发现当前线程曾经被中断过，那就把当前线程再中断一次**。

为什么这样做呢？

从上面的代码我们知道，即使线程在等待资源的过程中被中断唤醒，它还是会坚持不懈的抢锁，知道它抢到锁为止。也就是说，**它是不响应中断的**，仅仅是记录自己被人中断过。

最后，当它抢到锁返回了，发现自己曾经被中断过，它就再中断自己一次，将这个中断补上。

注意，中断对线程来说只是一个建议，一个线程被中断只是其中断状态被设为true，线程可以选择忽略这个中断，中断一个线程并不会影响线程的执行。

这里补充一点，在return interrupted处返回时并不是直接返回的，因为还有一个finally代码块：

```java
finally {
    if (failed)
        cancelAcquire(node);
}
```

这个执行的情况，其实我没太懂，这里failed为true的时候，只能子类实现tryAcquire抛出异常了，然后进行相关的善后工作。