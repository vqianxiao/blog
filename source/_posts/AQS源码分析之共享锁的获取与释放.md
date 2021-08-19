---
layout:    post
title:     AQS源码分析之共享锁的获取与释放
category:  JUC
description: AQS源码分析之共享锁的获取与释放
tags: 
    - JDK
    - JUC
    - AQS
date: 2021/08/19 22:37:10

---



#### 共享锁与独占锁的区别

共享锁和独占锁的最大区别在于，独占锁是独占的，排他的，因此独占锁中有一个`exclusiveOwnerThread`属性，用来记录当前持有锁的线程，当独占锁已经被某个线程持有时，其他线程只能等待它被释放时，才能去竞争锁，并且同一时刻只能有一个线程竞争锁成功。

但是对于共享锁而言，它是可以被多个线程同时持有。如果一个线程成功获取了共享锁，那么其他等待在这个共享锁上的线程就也可以尝试去获取锁，并且极有可能获取成功。

| 独占锁                                     | 共享锁                                           |
| ------------------------------------------ | ------------------------------------------------ |
| tryAcquire(int arg)                        | tryAcquireShared(int arg)                        |
| tryAcquireNanos(int arg,long nanosTimeout) | tryAcquireSharedNanos(int arg,long nanosTimeout) |
| acquire(int arg)                           | acquireShared(int arg)                           |
| acquireQueued(final Node node,int arg)     | doAcquireShared(int arg)                         |
| acquireinterruptibly(int arg)              | acquireSharedInterruptibly(int arg)              |
| doAcquireInterruptibly(int arg)            | doAcquireSharedInterruptibly(int arg)            |
| doAcquireNanos(int arg,long nanosTimeout)  | doAcquireSharedNanos(int arg,long nanosTimeout)  |
| release(int arg)                           | releaseShared(int arg)                           |
| tryRelease(int arg)                        | tryReleaseShared(int arg)                        |
| -                                          | doReleaseShared()                                |

除了最后一个共享锁的doReleaseShared()方法独占锁没有对应外，其他方法，共享锁和独占锁都是一一对应的。

在独占锁中，我们只有在获取独占锁的节点释放锁时，才会唤醒后继节点——这是合理的，因为独占锁只能被一个线程持有，如果它还没被释放，就没必要去唤醒它的后继节点。

但是在共享模式下，当一个节点获取到了共享锁，我们在获取成功后就可以唤醒后继节点，而不需要等到释放锁的时候再去唤醒。这是因为共享锁可以被多个线程同时持有，一个锁获取到了，则后继的节点都可以直接来获取。因此，在共享锁模式下，在获取锁和释放锁结束时，都会唤醒后继节点。所以doReleaseShared()方法与unparkSuccessor(h)方法无法直接对应。

#### 共享锁的获取

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

独占锁的获取

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

可以看到共享锁tryAcquireShared()返回的是一个int类型而独占锁tryAcquire()返回的是boolean类型的。

tryAcquireShared返回值：

- 如果该值小于0，表示当前线程获取共享锁失败
- 如果该值大于0，表示当前线程获取共享锁成功，并且接下来其他线程获取共享锁的行为很可能成功
- 如果该值等于0，表示当前线程获取共享锁成功，但是接下来其他线程尝试获取共享锁的行为会失败

因此只要返回值大于0，就表示获取共享锁成功。

tryAcquireShared由子类实现，先不看。

看下doAcquireShared方法，对应独占锁的acquireQueued。

```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    //独占锁这里是
    //addWaiter(Node.EXCLUSIVE)
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    //独占锁这里是
                    //setHead(node)
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

我在共享锁和独占锁不同的地方加了注释，可以看到，第一点不同是独占锁的acquireQueued调用的是addWaiter(Node.EXCLUSIVE)，而共享锁调用的是addWaiter(Node.SHARED)，表明该节点处于共享模式，两种模式的定义为：

```java
static final Node SHARED = new Node();
static final Node EXCLUSIVE = null;
```

这个模式被赋值给了节点的nextWaiter属性：

```java
Node(Thread thread, Node mode) {     // Used by addWaiter
    this.nextWaiter = mode;
    this.thread = thread;
}
```

我们知道，在条件队列中，nextWaiter是指向条件队列中的下一个节点的，它将条件队列中的节点穿起来，构成了单链表。但是在sync queue队列中，我们只用prev，next属性来串联节点，这样就形成了双向链表，nextWaiter属性在这里只起到一个标记作用，不会串联节点。Node SHARED = new Node() 这个节点并不属于sync queue，不代表任何线程，只是作为判断节点是否处于共享模式的依据（Node中isShared方法）：

```java
final boolean isShared() {
     return nextWaiter == SHARED;
 }
```

第二点不同在于获取锁成功后的行为，对于独占锁而言，是直接调用了setHead(node)方法，而共享锁调用的是setHeadAndPropagate(node,r)：

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);

    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

在这个方法内部，不进调用了setHead(node)方法，还在一定条件下调用了doReleaseShared()来唤醒后继的节点。这是因为在共享锁模式下，锁可以被多个线程所共同持有，既然当前线程已经拿到了共享锁了，那么就可以直接通知后继节点来拿锁，而不必等到锁被释放的时候再通知。这个doReleaseShared()方法在分析释放锁的时候再看。

#### 共享锁的释放

共享锁释放的方法是releaseShared(int arg) ：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

独占锁的释放的方法release(int arg)：

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

在独占锁模式下，由于头节点就是独占锁的节点，在它释放独占锁后，如果发现自己的waitStatus不为0，则它将唤醒它的后继节点。

在共享模式下，头节点就是持有共享锁的节点，在它释放共享锁后，它也应该唤醒它的后继节点，但是值得注意的是，我们在之前的setHeadAndPropagate方法中可能已经调用过该方法了，也就是说它可能被同一个头节点调用两次，也有可能在我们从releaseShared方法中调用它时，当前的头节点已经易主了。

看下doReleaseShared这个方法：

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

这个方法比较难理解。我们先从哪里调这个方法，谁调用这个方法，调用这个方法的目的是什么这些方面去分析。

**1.该方法有几处调用？**

这个方法一共有两处调用，一处是acquireShared末尾，当线程成功获取到共享锁后，在一定条件下调用该方法，一处是在releaseShared方法中，当线程释放共享锁的时候调用。

**2.调用该方法的线程是谁？**

在独占锁中，只有获的了锁的线程才能调用release释放锁，因此调用unparkSuccessor(h)唤醒后继节点的必然是持有锁的线程，该线程可看做是当前的头节点（虽然setHead方法中已经将头节点的thread属性设为null了，但是这个头节点曾经代表的就是这个线程）

在共享锁中，持有共享锁的线程可以是多个，这些线程都可以调用releaseShared方法释放锁。而这些线程想要获得共享锁，则他们必然曾经成为过头节点，或者就是现在的头节点。因此，如果在releaseShared方法中调用的doReleaseShared，可能此时调用方法的线程已经不是头节点所代表的线程了，头节点可能已经被易主好几次了。

**3.调用该方法的目的是什么？**

这个方法的作用是在当前共享锁是可获取状态时，唤醒head节点的下一个节点。这一点看起来和独占锁似乎一样，但是它们有一个重要的差别是，在共享锁中，当头节点发生变化时，会回到循环中再立即唤醒head节点的下一个节点的。也就是说，在当前节点完成唤醒后继节点的任务之后将要退出时，如果发现被唤醒节点的后继节点已经成为新的头节点，则会立即出发唤醒head节点的下一个节点的操作，直到在这个循环中头节点没有发生变化。

**4.退出该方法的条件是什么？**

该方法是一个自旋操作，只有满足h == head条件后的break可以退出这个方法。也就是在这次循环中，头节点没有发生变化。

假设队列中依次排列有

> dummy node ->A -> B -> C -> D

现在假设A拿到了共享锁，则它将成为新的dummy node

> dummy node(A) -> B -> C-> D

此时A线程会调用doReleaseShared，我们写作doReleaseShared[A]，在该方法中将继续唤醒后继的节点B，B很快获得了共享锁，成为新的头节点

> dummy node(B) -> C -> D

此时B也会调用doReleaseShared，我们写作doReleaseShared[B]，该方法将唤醒后继的节点C，但是在doReleaseShared[B]调用的时候，doReleaseShared[A]还没运行结束，当它运行到if(h == head)时，发现头节点已经变了，所以它将继续回到for循环中，doReleaseShared[B]运行到判断的时候，也进入到for循环中。

这时这里就形成了一个“调用风暴”，大量线程在同时执行doReleaseShared，这将加快后继节点的唤醒速度，提升执行效率。同时内部的CAS又保证了多个线程唤醒同一个节点时，只有一个线程可以成功。

如果doReleaseShared[A]执行结束时，节点B还没有成为新的头节点，那么doReleaseShared[A]就会结束退出，这样也不会影响A唤醒了B线程的结果，最终等待共享锁的线程都将执行。

这里的“调用风暴”其实是一个优化操作，因为我们在执行到该方法末尾的时候，unparkSuccessor基本上已经被调用过了，然后因为是共享锁，所以被唤醒的后继节点也极有可能已经获取到了共享锁，成为新的head节点，当它成为head节点后，它可能还是要在setHeadAndPropagate方法中调用doReleaseShared唤醒它的后继节点。

```java
if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h); 
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
}
```

第一个if比较好理解，如果当前头节点ws值为SIGNAL，则说明后继节点需要被唤醒，这里使用CAS先改ws状态改为0，并且让成功的那个线程执行unparkSuccessor，这样也保证了unparkSuccessor只会被执行一次。

这个第二个if。是**当前队列最后一个节点变成了头节点**，为什么是这种情况呢？因为每次新加入一个节点，都会把自己的前驱节点的waitStatus改为SIGNAL。

这个if描述了一个极其严苛且短暂的状态：

1.首先，队列里有至少两个节点 可以看最外层的判断条件 if(h != null && h != tail)

2.要执行到else if语句，说明头节点是刚刚成为头节点的，它的waitStatus还是0，尾节点是在这之后加进来的，它需要执行shouldParkAfterFailedAcquire，将它的前驱节点（头节点）的waitStatus改为SIGNAL，但是目前这个修改操作还没有来得及执行。这种情况使得可以进入第二个if的前半段 ws == 0

3.紧接着要满足!compareAndSetWaitStatus(h, 0, Node.PROPAGATE)这一条件，说明此时头节点的waitStatus已经不是0了，说明之前那个没有来得及执行的在shouldParkAfterFailedAcquire将前驱节点的waitStatus值修改为SIGNAL的操作现在执行完了

所以第二个if的&&连接了两个不一致的状态，分别对应了shouldParkAfterFailedAcquire的`compareAndSetWaitStatus(pred, ws, Node.SIGNAL)`执行成功前和执行成功后，因为doReleaseShared和shouldParkAfterFailedAcquire是可以并发执行的，所以这一条件是有可能满足的，只是满足条件非常严苛，可能只是一瞬间的事。

#### 总结

- 共享锁的调用框架和独占锁很相似，它们最大的不同在于获取锁的逻辑——共享锁可以被多个线程同时持有，而独占锁同一时刻只能被一个线程持有
- 由于共享锁同一时刻可以被多个线程持有，因此当头节点获取到共享锁时，可以立即唤醒后继节点来竞争锁，而不必等到释放锁的时候。因此共享锁触发唤醒后继节点的行为可能有两处，一处在当前节点成功获得共享锁后，一处在当前节点释放共享锁后。