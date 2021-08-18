---
layout:    post
title:     死磕ReentrantLock源码（一）
category:  JUC
description: 死磕ReentrantLock源码（一）
tags:
- JDK
- JUC
date: 2021/08/17 12:08:10

---

锁从不同的维度去区分，可以分成很多中类别的锁。比如可以根据一个线程中的多个流程能不能获取同一把锁来区分可重入锁和不可重入锁，可以根据多个线程能不能共享同一把锁来区分共享锁和排他锁可以根据多个线程竞争锁时要不要排队来区分公平锁和非公平锁。可重入锁就是一个线程中多个流程能获取同一把锁，不可重入锁就是一个线程中多个流程不能获取同一把锁。共享锁就是多个线程能共享同一把锁，排他锁就是锁只能被一个线程持有的锁。多个线程竞争锁时要排队就是公平锁，先尝试插队插队失败再排队的就是非公平锁。

我想学习ReentrantLock主要目的就是为了知道它是怎么支持可重入的吧。ReentrantLock类中有Sync对象，这个对象继承了AbstractQueuedSynchronizer，也就是AQS（队列同步器）。AQS为线程的同步和等待等操作提供一个基础模板类。虽然为抽象类，但是没有抽象方法，将需要子类重写的方法定义成protect方法，将默认实现为抛出`UnsupportedOperationException`异常，如果子类用到，没有重写，将会抛出异常，如果子类没用用到，则不需要做任何操作。尽可能多的实现可重入锁，读写锁同步器所有需要的功能。队列同步器内部实现了线程的同步队列，独占或是共享的获取方式等，使其只需要少量的代码便可以实现目标功能。

一般来说，AQS的子类应以其他类的内部类的形式存在，然后使用代理模式调用子类和AQS本身的方法实现线程的同步。

本文源码基于JDK1.8。

java并发工具的设计套路，分为三个重要组成部分，分别是状态、队列、CAS。

- 状态：一般是一个state属性，是整个工具的核心，通常整个工具都是在设置和修改状态，很多方法的操作都依赖于当前状态是什么，由于状态是全局共享的，一般会使用volatile修饰，保证其修改的可见性。
- 队列：队列通常是一个等待的集合，大多数以链表形式实现。队列采用的是悲观锁思想，表示当前所等待的资源，状态或者条件短时间内可能无法满足。因此，它会将当前线程包装成某种数据结构，扔到一个等待队列中，当条件满足后，再从等待队列中取出。
- CAS：CAS操作是最轻量级的并发处理，通常对于状态的修改都会用到CAS操作，因为状态可能被多个线程同时修改，CAS操作保证了同一个时刻，只有一个线程能修改成功，从而保证了线程安全，CAS操作基本是由Unsafe工具类的`compareAndSwapXXX`来实现的。CAS采用的是乐观锁的思想，因此常常伴随着自旋，如果发现当前无法成功地执行CAS，则不断重试，直到成功为止，自旋的表现形式通常是一个死循环。

#### AQS核心实现

##### 状态

在AQS中，状态是由state属性来表示的

```java
private volatile int state;
```

该属性值表示了锁的状态，state为0表示锁没有被占用，state大于0表示当前已经有线程持有该锁。这里说大于0，是因为可能有可重入的情况，那时候state就是持有该锁的线程数。

独占锁的话，同一时刻，锁只能被一个线程持有。通过state变量是否为0，可以分辨当前锁是否被占用，但光知道锁是不是被占用是不够的，并不知道占用锁的线程是哪一个。在监视器锁中，我们用objectMonitor对象的_owner属性记录了当前拥有监视器锁的线程，而在AQS中，我们将通过exclusiveOwnerThread属性来记录占用锁的线程。

```java
private transient Thread exclusiveOwnerThread;
```

`exclusiveOwnerThread`属性的值就是当前持有锁的线程。

##### 队列

AQS中，队列的实现是一个双向链表，被称为`sync queue`，它表示所有等待锁的线程的集合，与synchronized中的wait set。

并发编程在线程拿不到锁的时候，通常是把当前线程包装成某种类型的数据结构扔到等待队列中。我把重要的部分粘出来

```java
static final class Node {
    //节点所带表的线程
    volatile Thread thread;
    
    //双向链表
    volatile Node prev;
    volatile Node next;
        
    //线程所处的等待锁的状态，初始化时，该值为0
    volatile int waitStatue;
    static final int CANCELLED =  1;
	static final int SIGNAL    = -1;
	static final int CONDITION = -2;
	static final int PROPAGATE = -3;
    
    //该属性用于条件队列或者共享锁
}
```

**tips** 在Node类中也有一个`waitStatus`，它表示了当前Node所代表的线程的等待锁的状态，在独占锁模式下，我们只需要关注`CANCELED`、`SIGNAL`两种状态即可。这里还有一个`nextWaiter`属性，它在独占锁模式下永远为null，仅仅起到一个标记作用，没有实际意义。

AQS怎么去使用这个队列呢，既然是双向链表，那么肯定要有一个头节点和尾节点。

```java
//头结点，不代表任何线程，只是一个哑节点
private transient volatile Node head;
//尾节点，每一个请求锁的线程都会被加到尾节点
private transient volatile Node tail;
```

在AQS中的队列是一个CLH队列，它的head节点永远是一个哑节点(dummy node)，它不代表任何线程（某些情况下可以看做是代表了当前持有锁的线程），**因此head所指向的Node的thread属性永远是null**。只有从次节点往后的所有节点才代表了所有等待锁的线程。也就是说，在当前线程没有抢到锁被包装成Node扔到队列中时，即使队列是空的，它也会排在第二个，我们会在它的前面新建一个dummy节点。为了便于描述，除去head节点的队列称作等待队列，这个队列中的节点代表等待锁的线程。

![]((/blog/images/aqs/syncqueue.png))

结合图片，总结下Node节点各个属性的含义：

- thread：当前Node所代表的线程
- waitStatus：表示节点所处的等待状态，共享锁模式状态下只需关注三种状态：`SIGNAL`、`CANCELED`、`初始状态(0)`
- prev next：节点的前驱和后继
- nextWaiter：仅作为标记，值为null表示当前处于独占锁模式

##### CAS

CAS操作大多数都是用来改变状态的，在AQS中也不例外。我们一般在静态代码块中初始化需要CAS操作的属性的偏移量。

```java
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;

    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));

        } catch (Exception ex) { throw new Error(ex); }
    }
```

从静态代码块可以看出，CAS操作主要针对五个属性，包括AQS的3个属性state、head、tail和Node对象的两个属性waitStatus、next。说明这5个属性基本是会被多个线程同时访问的。

看下实际CAS操作有哪些

```java
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
private final boolean compareAndSetHead(Node update) {
    return unsafe.compareAndSwapObject(this, headOffset, null, update);
}
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}
private static final boolean compareAndSetWaitStatus(Node node, int expect,int update) {
    return unsafe.compareAndSwapInt(node, waitStatusOffset, expect, update);
}
private static final boolean compareAndSetNext(Node node, Node expect, Node update) {
    return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
}
```

然后就是不断自旋调用CAS操作来保证操作成功了。

ReentrantLocak有公平锁和非公平锁两种实现，默认为非公平锁，体现在默认构造函数中。

```java
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

 	// 获取锁
    public void lock() {
        sync.lock();
    }
```

FairSync继承自Sync，而Sync继承自AQS，ReentrantLocak获取锁的逻辑是直接调用了FairSync或者NonfairSync的逻辑。

#### 获取锁

看下ReentrantLock里的NonFairSync代码

```java
	static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        final void lock() {
            if (compareAndSetState(0, 1))
                //这个方法下面贴出来了 其实就是把当前线程赋值给exclusiveOwnerThread
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
```

可以看到在执行lock()的时候，上来就先假设没有锁，先去通过CAS修改锁的状态，如果修改成功，将当前线程赋值给exclusiveOwnerThread，也就是表明当前线程持有了锁。否则就去尝试获取锁，进入acquire(1)。

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

#### acquire

acquire定义在AQS类中，描述了获取锁的流程。包含了四个方法调用：

1.tryAcquire(arg)

该方法由继承AQS的子类实现，为获取锁的具体逻辑。

2.addWaiter(Node mode)

该方法由AQS实现，负责在获取锁失败后调用，将当前请求锁的线程包装成Node扔到sync queue中去，并返回这个Node。

3.acquireQueued(final Node node,int arg)

该方法由AQS实现，主要对上面刚加入队列的Node不断尝试以下两种操作之一：

- 在前驱节点就是head节点时，继续尝试获取锁
- 将当前线程挂起，使CPU不再调度它

4.selfInterrupt

该方法由AQS实现，用于中断当前线程。由于在整个抢锁过程中，我们都是不响应中断的。那如果在抢锁过程中发生了中断怎么办呢，总不能假装没看见。AQS的做法就是简单的记录有没有发生过中断，如果返回的时候发现曾经发生过中断，则在退出acquire方法之前，就调用selfInterrupt自我中断一下，就好像将这个发生在抢锁过程中的中断“推迟”到抢锁结束以后再发生一样。

从上面的介绍可以看出，除了获取锁的逻辑tryAcquire(arg)由子类实现之外，其余方法均由AQS实现。

tryAcquire在ReentrantLock.Sync类中实现，根据调用链追踪下来，发现就是调用ReentrantLock.Sync中的nonfairTryAcquire()方法。

```java
    final boolean nonfairTryAcquire(int acquires) {
final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
        if (compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
        }
        }
        else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
        throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
        }
        return false;
        }
```

这里可以看到，如果state==0继续CAS去改变state的值为1然后更新持有锁的线程。当当前线程与持有锁的线程是同一个线程时，将state和acquires相加，得到一共加了多少次锁，这个很关键，因为需要相同次解锁这个资源才可以被别的线程持有。否则，直接返回false。

acquire方法中还调用了acquireQueued()方法，还有addWaiter()方法。`Node.EXCLUSIVE`参数表明这是独占模式。

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //当队列尾部不为空的时候 并且cas将队列尾部node更新为当前node 将node插入等待队列的尾部
        if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
        pred.next = node;
        return node;
        }
        }
        enq(node);
        return node;
        }

private Node enq(final Node node) {
        for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
        if (compareAndSetHead(new Node()))
        tail = head;
        } else {
        node.prev = t;
        if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
        }
        }
        }
        }
```

再来看acquireQueued()，这个方法比较复杂，主要对上面刚加入队列的Node不断尝试2种操作：

1.在前驱节点就是head节点的时候，继续尝试获取锁

2.将当前线程挂起，让CPU不再调度它

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
        boolean interrupted = false;
        for (;;) {
final Node p = node.predecessor();
        //p == head 说明当前node已经是第一个节点了 所以再尝试获取一下锁 拿到锁，将node赋值给head 返回false
        if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
        }
        //shouldParkAfterFailedAcquire这个放回返回true 将会调用parkAndCheckInterrupt进入阻塞状态
        if (shouldParkAfterFailedAcquire(p, node) &&
        parkAndCheckInterrupt())
        interrupted = true;
        }
        } finally {
        if (failed)
        cancelAcquire(node);
        }
        }

//获取前一个节点
final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
        throw new NullPointerException();
        else
        return p;
        }

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //获取前置节点的waitStatus
        //CANCELLED 1 取消
        //SIGNAL -1 表明后续线程需要运行 indicate successor's thread needs unparking
        //CONDITION -2 处于等待条件 indicate thread is waiting on condition
        //PROPAGATE -3 indicate the next acquireShared should unconditionally propagate
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
        if (ws > 0) {
        //前面的节点被取消 跳过已经取消等待锁的节点 往前找直到找到排队的节点
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
        node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
        } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        //waitStatus必须为0或者-3。表明我们需要一个信号，但是不要阻塞。调用者需要重试来确认暂停前无法获得锁
        //把前一个节点状态赋值成SIGNAL 让线程重试获取锁，避免不必要的阻塞
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
        }

//将线程挂起然后等待唤醒并返回当前线程是否被中断
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
        }
```

这里需要注意SIGNAL这个状态不是为线程自己设置的，是为前一个节点设置的。

#### 获取锁总结

1.先尝试CAS修改state，如果改成功，说明当前线程获取锁成功，然后将持有锁的线程更新成当前线程（exclusiveOwnerThread）。如果没有修改成功，调用acquire()

2.在acquire()方法中，再次通过tryAcquire()获取锁，获取失败，会调用addWaiter方法，将当前等待的线程包装成Node，并成功添加到队列的末尾，这个操作由enq()方法来保证，enq()方法同时还负责在队列为空时初始化队列。

3.acquireQueued方法用于Node成功入队后，继续尝试获取锁（取决于Node的前驱节点是不是head），或者将线程挂起。

4.shouldParkAfterFailedAcquire方法用于保证当前线程的前驱节点的waitStatus属性值为SIGNAL，从而保证了自己挂起后，前驱节点会负责在合适的时候唤醒自己。

5.parkAndCheckInterrupt方法用于挂起当前线程，并检查中断状态。

6.如果最终成功获取了锁，线程会从lock()方法返回，继续往下执行，否则，线程会阻塞等待。