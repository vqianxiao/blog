---
layout:    post
title:     AQS源码分析之Condition接口的实现
category:  JUC
description: AQS源码分析之Condition接口的实现
tags: 
    - JDK
    - JUC
    - AQS
date: 2021/09/02 15:05:10

---

Condition接口是用来做控制线程的执行流程的。类似于监视器锁的wait/notify机制。可以先看下Condition提供了哪些方法。Condition接口一共定义了7个方法，根据方法名方法的用途我们也能猜个大概。

| Object                            | Condition                              | 区别                         |
| --------------------------------- | -------------------------------------- | ---------------------------- |
| void wait()                       | void await()                           |                              |
| void wait(long timeout)           | long awaitNanos(long nanosTimeout)     | 时间单位不同，返回值         |
| void wait(long timeout,int nanos) | boolean await(long time,TimeUnit unit) | 时间单位同，参数类型，返回值 |
| void notify()                     | void signal()                          |                              |
| void notifyAll()                  | void signal                            |                              |
| -                                 | void awaitUninterruptibly()            | Condition独有                |
| -                                 | boolean awaitUntil(Date deadline)      | Condition独有                |

类比下wait/notify机制：

1.调用wait方法的线程必须是已经进入了同步代码块的线程，也就是获得了监视器锁，那么调用await方法的线程也必须是获得了lock锁。

2.调用wait方法的线程会释放已获得的监视器锁，进入当前监视器锁的等待队列（wait set）中，调用await方法的线程也会释放已经获得的lock锁，进入到当前Condition对应的条件队列中。

3.调用监视器锁的notify方法会唤醒等待在该监视器锁上的线程，这些线程将开始参与锁竞争，并在获得锁后，从wait方法处恢复执行。调用Condition的signal方法会唤醒对应条件队列中的线程，这些线程将开始参与锁竞争，并在获得锁后，从await方法处开始恢复执行。

先来看下Condition怎么使用，这里看一下官方给的例子：

```java
class BoundedBuffer {
    final Lock lock = new ReentrantLock();
    final Condition notFull = lock.newCondition();
    final Condition notEmpty = lock.newCondition();

    final Object[] items = new Object[100];
    int putptr, takeptr, count;

    // 生产者方法，往数组里面写数据
    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await(); //数组已满，没有空间时，挂起等待，直到数组“非满”（notFull）
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            notEmpty.signal(); 
        } finally {
            lock.unlock();
        }
    }

    // 消费者方法，从数组里面拿数据
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await(); // 数组是空的，没有数据可拿时，挂起等待，直到数组非空（notEmpty）
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

这个例子是一个生产者-消费者模型，在同一个lock上，有两个条件队列，当数组满的时候，put方法在notFull条件上等待，直到数组不满的时候。当数组为空时，take方法在notEmpty条件上等待，直到数组不为空的时候。

我们知道，在AQS中所有等待锁的线程会被包装成Node扔到一个同步队列中去，那么因为条件不满足继续执行下去而调用await方法的线程应该也是放到一个队列中去了，而一个lock中又可以有多个Condition，所以我们猜测应该每个Condition内部维护一个条件队列，这样就可以互相不影响了。

但是和等待锁的同步队列不同的是，同步队列是一个双向链表，nextWaiter并没有用来串联链表，而是用了prev，next属性。条件队列是一个单向链表，用nextWaiter来串联链表。

### Condition实现

AQS中Condition的主要实现是ConditionObject，在ConditionObject中有两个重要的属性，一个是firstWaiter，一个是lastWaiter。

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

然后我们看await()方法。

##### await()

```java
public final void await() throws InterruptedException {
    //如果调用await()前已经被中断，则直接抛出InterruptedException
    if (Thread.interrupted())
        throw new InterruptedException();
    //将当前线程加到等待队列中
    Node node = addConditionWaiter();
    //释放当前线程所占用的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //如果当前node不在同步队列中，则直接将线程挂起
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        //检查线程被唤醒原因，是signal还是中断
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //尝试去获取锁 然后执行后续逻辑
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

源码中对一些代码也进行了注释，流程基本上就是这样。关于被唤醒之后的分析，我们后面再来看。

##### addConditionWaiter

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    //如果尾节点被cancel了，则先遍历整个链表，清除所有被cancel的节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //当前节点入条件队列
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

这里看到代码中没有任何加锁操作，那么这个会存在并发问题吗？不存在，因为能进入到await方法中的线程一定是已经获得了锁的，而能获得锁的线程只有一个，也就不会存在并发，所以不需要加锁操作。在这个方法中，我们就是简单的将当前线程封装成Node加到条件队列的末尾。

这里面有三个点需要注意一下：

1. 节点加入sync queue是waitStatus的值是0，但节点加入condition queue是waitStatus的值为Node.CONDITION（-2）。
2. sync queue的头节点是虚节点，如果队列为空，会先创建一个dummy节点，再创建一个代表当前节点的Node添加在dummy节点的后面。condition queue没有dummy节点，初始化时，直接将firstWaiter和lastWaiter直接指向新建的节点。
3. sync queue是一个双向队列，在节点入队后，需要同时修改当前节点的前驱和后继。在condition queue中，节点入队，只需要修改前驱节点的nextWaiter，也就是说条件队列是当单项队列使用的。

如果入队的时候，发现尾节点已经取消等待了，那么就不应该把新的node放到取消的尾节点后面，此时调用unlinkCancelledWaiters来剔除那些已经取消等待的线程：

```java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

这个方法就是从头部开始遍历队列，剔除waitStatus不是Node.CONDITION的节点。

##### fullyRelease

节点被成功添加到队列末尾后，将调用fullyRelease来释放当前线程所占用的锁：

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

这个方法中，通过release方法去释放锁。但是这里需要注意，这里对于可重入锁而言，是一次性释放所有的锁，因为getState()拿到的是重入的次数。然后这里还需要注意，这里会抛出IllegalMonitorStateException异常，这是因为当前线程可能不是持有锁的线程，前面说执行await方法的时候一定是拿到锁的线程才能调用await方法。但是我们并没有去判断当前拿到锁的线程是不是exclusiveOwnerThread这个线程。这个检测其实是在AQS子类实现tryRelease方法来做的。ReentrantLock对tryRelease方法的实现如下：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

当发现当前线程不是持有锁的线程时，就会进入fullyRelease的finally语句块中，将当前node的waitStatus改为Node.CANCELLED，这也是为什么添加新节点的时候，都会检查尾节点是否已经被取消了。因为会抛出IllegalMonitorStateException异常。

在当前线程的锁被释放后，就可以调用LockSupport.park(this)把当前线程挂起，等待signal了。但是源码中还调用了isOnSyncQueue来检查当前线程不在sync queue中。

```java
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;

    return findNodeFromTail(node);
}
```

```java
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

当前线程应该在一个独立的和sync queue无关的队列中，怎么会出现在sync queue中呢？这个要结合signal方法来看。

##### signalAll

调用signalAll方法的线程本身时已经持有了锁，并且准备释放锁的线程。在条件队列里的线程是已经在对应条件上挂起了，等待被signal唤醒，然后去争夺。

与调用notify时线程必须时已经持有了监视器锁类似，在调用condition的signal方法时，线程必须是已经持有了lock锁：

```java
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}
```

isHeldExclusively这个方法是检查当前线程是不是持有锁的线程，这个方法通常由AQS的子类来实现。ReentrantLock对该方法的实现为：

```java
        protected final boolean isHeldExclusively() {
          return getExclusiveOwnerThread() == Thread.currentThread();
        }
```

拿到firstWaiter判断条件队列是否为空，如果条件队列不为空，则调用doSignalAll方法：

```java
private void doSignalAll(Node first) {
    //清空整个队列
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

通过lastWaiter = firstWaiter = null;将整个队列清空掉，然后通过循环将原来的队列的节点拿出来，通过transferForSignal方法将线程一个一个放到sync queue末尾。注意这里是一个一个放进去的，可以看下transferForSignal：

```java
final boolean transferForSignal(Node node) {  
    //如果这个节点在signal前被取消，那么直接返回
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    //节点插入到sync queue中
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

在transferForSignal方法中，先通过CAS操作将当前节点的waitStatus状态由CONDITION设为0，修改失败，则认为该节点已经被取消了，直接返回，操作下一个节点。如果修改成功插入到同步队列中去，这里需要注意enq返回的是node的前驱节点。sync queue中的节点需要靠前驱节点唤醒，所以需要把前驱节点waitStatus设为Node.SIGNAL。这里只要前驱节点被取消或者无法将前驱节点状态修改为Node.SIGNAL，那我们就将Node所代表的线程唤醒，但是这个不意味着lock是可以被获取的，只是唤醒这个线程去获取锁，如果获取失败，继续进入同步队列等待。

总结一下signalAll方法：

1. 清空条件队列（只是把lastWaiter、firstWaiter设为null，队列中节点的关系还存在）
2. 将条件队列中的头节点取出，将它和后面的节点断开连接，然后调用transferForSignal方法
3. 判断该节点处于被取消状态，直接跳过该节点（该节点会被GC回收）。节点状态自由，则通过enq方法，将节点放到sync queue的末尾
4. 根据当前节点的前驱节点的waitStatus状态判断是否要唤醒当前节点，如果前驱节点被取消，或者前驱节点设置SIGNAL状态失败，直接唤醒该节点的线程
5. 重复2到4的过程，直到整个条件队列中的节点都被处理完成

##### signal

signal只是唤醒一个节点，对于AQS的实现来说，只是唤醒条件队列中第一个没有被取消的节点。

```java
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

isHeldExclusively方法已经分析过，检查当前线程是不是持有锁的线程。然后看doSignal方法：

```java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

这个看到是一个循环，也是为了找队列中第一个没有被取消的节点，然后将这个节点放到同步队列中去。也是将first取出来，然后将nextWaiter设置为null，调用transferForSignal方法，这个方法的返回值将决定循环是否继续。如果返回true，那么while循环的条件就不满足了，所以只会唤醒一个线程。

唤醒已经讲完了，我们再来看await方法。因为唤醒的时候，会让线程回到原来阻塞的地方继续执行。

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); 
        // 这里就是阻塞回来的位置
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

但是线程被唤醒，并不知道是什么原因被唤醒的。但是既然唤醒了，那么就需要acquireQueued方法进行“阻塞式”获取锁，抢到锁就返回，抢不到锁就继续被挂起。当await方法返回时，当前线程一定已经持有了lock锁。

如果从线程被唤醒，到线程获取到锁这段过程中发生过中断，应该如何处理呢？

中断对当前线程只是个建议，有当前线程决定怎么对其做出处理。acquireQueued方法对中断是不响应的，只是简单的记录抢锁过程中的中断状态，并在抢到锁后将这个中断状态返回，交给上层调用函数处理，此时也就是await方法。await方法如何处理这个中断，**取决于这个中断发生时，线程是否已经被signal**。

在调用await方法时，就会检查当前线程的中断状态，如果线程还未阻塞就被中断，那么直接抛出InterruptedException。如果中断发生时，线程没有被signal过，那么等待中的线程就会被打断，也就是一个意料之外的唤醒，所以线程被唤醒后，需要抛出InterruptedException，表示当前线程因为中断而被唤醒。如果中断发生时，当前线程已经被signal过了，它就已经正常地从condition queue中唤醒了，那么这个中断来的就比较晚了，因为已经是正常唤醒之后才来的中断，那么就在await方法中补一下这个中断，调用Thread.interrupted()方法检测是否发生中断时也会将中断状态清除，因此如果忽略中断，则应该在await方法将线程恢复成原来的样子。

在await方法中，用变量interruptMode记录中断事件：

1. 0：没有中断发生
2. 1：REINTERRUPT 表示中断发生在signal之后，需要再次自我中断
3. -1:THROW_IE 表示中断发生在signal之前，需要抛出InterruptedException

acquireQueued这个方法两个参数，我们前面说了释放锁的时候是释放所有的可重入锁，现在加锁的时候也是原来释放多少，这次就加多少锁。unlinkCancelledWaiters这个方法已经讲过了，就是遍历链表，然后把waitStatus不为CONDITION的节点从队列中移除。reportInterruptAfterWait是用来汇报中断的：

```java
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

如果interruptMode=THROW_IE，则直接抛出异常，报告等待的时候被中断异常唤醒。

来总结一下：

**中断发生在signal之前**

1. 线程因为中断被唤醒，通过checkInterruptWhileWaiting方法调用transferAfterCancelledWait方法来确认线程的waitStatus的值为Node.CONDITION，说明线程没有被signal
2. 修改线程的waitStatus为0，通过enq(node)将其添加到sync queue中
3. 线程获取锁，如果获取不到，则被再次挂起
4. 线程获取到锁后，调用unlinkCancelledWaiters方法将自己从条件队列中移除，该方法顺便移除其他取消等待的锁
5. 通过reportInterruptAfterWait抛出InterruptedException

一个调用await方法挂起的线程在被中断后不会立即抛出InterruptedException，而是进入同步队列去竞争锁，争不到，挂起，如果争到锁了，从同步队列和条件队列移除，抛出InterruptedException。

**中断发生在signal之后**

这种情况其实就是线程被唤醒后，然后产生的中断，并没有影响到我们线程正常的阻塞到被signal唤醒的流程，所以，我们并不需要抛出InterruptedException异常，我们可以忽略这个中断，在await方法返回的时候，将这个线程重新中断一下，对忽略的中断进行一个补偿。

##### await总结

我们对整个await方法进行一个总结：

1. 进入await()时必须已经持有了锁
2. 离开await()时必须已经持有了锁
3. 调用await()会使当前线程被封装成Node扔进条件队列，并且释放所有持有的锁
4. 释放锁后，当前线程被挂起，等待signal或者中断唤醒线程
5. 线程被唤醒后，进入同步队列进行抢锁
6. 线程在抢到锁之前发生过中断，则根据中断发生在signal之前还是之后记录中断模式，体现在interruptMode
7. 线程在抢到锁后进行售后工作（离开条件队列，处理中断异常）

##### awaitUninterruptibly

中断和signal都可以把线程唤醒，但是中断是一种异常唤醒，唤醒以后线程发现等待的条件并为满足，还是需要将线程挂起。当不希望await方法被中断时，可以使用awaitUninterruptibly方法：

```java
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true;
    }
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}
```

可以看到就算中断唤醒了线程，也仅仅是记录interrupted状态为true，然后继续阻塞，等到signal才会唤醒线程去获取锁，当发生中断，不管什么时候的中断，都是最终补偿中断一次，并且不会抛出InterruptedException

##### awaitNanos

前面的await和awaitUninterruptibly，它们在抢锁的过程中都是阻塞式的，一直抢到锁才能返回，否则线程一直挂起，这样的话就是线程如果长时间抢不到锁，就会一直被阻塞，因此有时候更需要带超时机制的去抢锁，和wait(long timeout)很像。看下源码：

```java
public final long awaitNanos(long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    final long deadline = System.nanoTime() + nanosTimeout;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (nanosTimeout <= 0L) {
            transferAfterCancelledWait(node);
            break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        nanosTimeout = deadline - System.nanoTime();
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return deadline - System.nanoTime();
}
```

这个方法基本上和await方法一样，只是多了超时时间的处理。上面这段代码的意思是，如果超时时间没到，那么就将线程挂起，超过等待时间了，将线程从条件队列转移到同步队列中。这里对超时时间很短的时间做了一个优化，当超时时间小于spinForTimeoutThreshold(1000L ns)时，通过自旋的方式等待唤醒，而不是将线程挂起。

wait(0)表示无限期等待，那么如果awaitNanos我们传0是一样的吗？

```java
if (nanosTimeout <= 0L) {
    transferAfterCancelledWait(node);
    break;
}
```

上面代码中已经有这段了，我这里单独提出来再来看下。这里可以看到当nanosTimeout<=0的时候，就会将线程从条件等待队列转移到同步队列中，并不会被挂起，也不需要signal。

##### await(long time, TimeUnit unit)

这个方法就是在awaitNanos方法基础上多了对超时时间单位的设置，但是在内部实现上还是会把时间转成纳秒去执行

```java
public final boolean await(long time, TimeUnit unit)
    throws InterruptedException {
    long nanosTimeout = unit.toNanos(time);
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    final long deadline = System.nanoTime() + nanosTimeout;
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (nanosTimeout <= 0L) {
            timedout = transferAfterCancelledWait(node);
            break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        nanosTimeout = deadline - System.nanoTime();
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;
}
```

awaitNanos(long nanosTimeout)的返回值是剩余的超时时间，如果该值大于0，说明超时时间还没到，则说明该返回是由signal行为导致的，而await(long time,TimeUnit unit)的返回值是transferAfterCancelledWait(node)的值决定的，调用该方法时，如果node还没有被signal过则返回true，否则返回false。其实这个方法就等价于调用awaitNanos(unit.toNanos(time)) > 0。

### awaitUntil(Date deadline)

awaitUntil方法和上面几个方法类似，只不过这个参数是一个绝对时间

```java
public final boolean awaitUntil(Date deadline)
    throws InterruptedException {
    long abstime = deadline.getTime();
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (System.currentTimeMillis() > abstime) {
            timedout = transferAfterCancelledWait(node);
            break;
        }
        LockSupport.parkUntil(this, abstime);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;
}
```

这个方法仅仅就是时间用成了绝对时间，并且没有使用spinForTimeoutThreshold进行自旋优化，因为调用这个方法，目的就是设一个较长的等待时间，不然上面的其他方法用起来更方便一些。