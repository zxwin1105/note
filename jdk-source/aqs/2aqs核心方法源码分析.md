        之前了解了AQS基础内容，分析了AQS中核心的方法，之后会对这写方法源码进行分析。

# 一、aquire&tryAquire

        之前介绍过tryAquire()是一个钩子方法，需要子类实现，这里我们借助ReentrantLock类来实现理解，稍后会对ReentrantLock类做简单的介绍。

## 1.1 ReentrantLock简单了解

        ReentrantLock类借助AQS实现了一套同步锁机制，属于独占锁，提供了公平锁与非公平锁的实现。ReentrantLock与synchronized关键字类似，ReentrantLock提供了lock()获取锁，unLock()释放锁，lock()方法内部是调用AQS的aquire()实现的；unLock()方法是调用AQS的release()实现的。

        ReentrantLock内部类关系：ReentrantLock有3个内部类:

- Sync：继承了AQS类重写了钩子方法，没有重写tryAquire()方法，而是交给两个子类实现，这两个子类都实现了tryAquire()方法，区别在于一个是公平锁，一个是非公平锁。

- FairSync：公平锁的实现

- NonfairSync：非公平锁的实现

        对于AQS源码分析，我们借助与NonfairSync来分析。

## 1.2 tryAquire源码分析

        之前说过tryAquire()是尝试获取锁，由AQS模板方法aquire()调用该方法，以下是ReentrantLock&NonfairSync中的tryAquire()

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

        nonfairTryAcquire()由Sync提供

```java
// 非公平机制，尝试获取锁 acquires=1
final boolean nonfairTryAcquire(int acquires) {
    // 获取当前尝试获取锁的线程
    final Thread current = Thread.currentThread();
    // 获取aqs.state状态
    int c = getState();
    // state没有被其他线程修改过，锁空闲
    if (c == 0) {
        // cas方式尝试修改state，修改成功后意味着获取到了        
        if (compareAndSetState(0, acquires)) {

            // 将aqs的工作线程设置为当前线程，当前线程持有锁        
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 可重入锁的实现，state被修改过，但是工作线程是当前线程，则是锁重入情况
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires; // state值表示锁重入的次数
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false; // 尝试获取锁失败
}
```

        从上述方法中可以看出，主要是对state的操作，主要功能如下：

1. 判读锁状态（state值），如果锁空闲，则尝试获取锁，获取成功后将aqs工作线程设置为当前线程，返回；如果获取锁失败返回false。

2. 如果锁被占用，则判断工作线程是否为当前线程，如果是，则进行锁重入操作。

## 1.3 aquire源码分析

```java
public final void acquire(int arg) {
    // 调用子类的tryAcquire()尝试获取锁，如果获取失败，继续执行if条件
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
        selfInterrupt(); 
}
```

        acquire()方法，主要功能如下：

1. 先调用子类的tryAcquire()方法，如果尝试获取锁成功，直接返回，如果获取失败，继续执行。

2. 尝试获取锁失败后，会调用addWaiter()创建一个Node节点封装当前线程，将Node加入到同步队列

3. node在同步队列的操作，阻塞线程，中断机制等功能，acquireQueued

4. 如果在获取锁的过程中，线程被中断，那么在获取锁成功后，线程会自己中断 selfInterrupt()。

        

### 1.3.1 addWaiter()方法

```java
private Node addWaiter(Node mode) {
    // 创建一个Node节点，封装当前线程
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 尝试将node快速入队，可能会失败（存在多个线程竞争关系）
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 尝试入队失败，则会一直尝试入队，直到成功。enq还有初始化队列的职责
    enq(node);
    return node;
}
```

        addWaiter()方法，主要功如下：

1. 创建Node节点，封装当前线程

2. 如果队列不为空，会尝试将node加入到同步队列尾部

3. 如果队列为空，获取node尝试加入到同步队列失败，调用enq()，enq()是职责就是初始化同步队列，且将Node入队

        

        enq()方法实现如下：

```java
private Node enq(final Node node) {
    // cas失败后，会继续尝试
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize 初始化队列
            if (compareAndSetHead(new Node()))
                tail = head;
        } else { // 注意这里入队的操作，非原子操作，在正向遍历整个sycn队列时需要注意t.next=node未执行，导致遍历不到所有节点
            // 1. node的prev指针指向t,
            node.prev = t;
            // 2.使tail指针指向node
            if (compareAndSetTail(t, node)) {
                // 3. t的next指针指向node,完成入队
                t.next = node;
                return t;
            }
        }
    }
}
```

        enq()方法比较简单，这里不做总结，需要注意的是入队操作并不是一个原子操作，

        

### 1.3.2 acquireQueued()方法

        分析完addWaiter()方法，会返回一个Node节点，此时节点已经加入到了同步队列，**且node.waitStatus = 0**。接下来会进行执行aquireQueued()方法，该方法功能较多，比较复杂，方法的功能主要包括，中断监测，尝试获取锁，修改线程状态，阻塞线程。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 是否被中断标识
        boolean interrupted = false;
        for (;;) {
            // 获取node的前驱节点
            final Node p = node.predecessor();
            // 如果前驱节点是head，会再次尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 如果获取锁成功，则将当前节点线程信息清空，将head执行当前节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 如果前驱节点不是head
            // shouldParkAfterFailedAcquire 更新节点状态，返回值表示线程是否需要被阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true; // 线程被打断，设置返回值
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}        
```

        方法功能分析：

1. 如果node时同步队列的第一个节点（非head）则尝试获取锁，如果获取成功将工作线程设置为当前线程，将node移出同步队列，返回中断状态。

2. 如果尝试获取锁失败，则先执行shouldParkAfterFailedAcqure()方法，该方法修改node.waitStatus，并判断线程是否需要被阻塞。

3. 如线程需要被阻塞则调用parkAndCheckInterrupt()阻塞线程，线程被唤醒时该方法会检查线程中断状态。

        

        shouldParkAfterFailedAcqure()方法源码

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;                
    // 如果node前驱节点waitStatus=-1，说明node前存在节点需要被唤醒，node则直接阻塞
    if (ws == Node.SIGNAL) //SIGNAL = -1
        /*
         * This node has already set status asking a releas
         * to signal it, so it can safely park.
         * node的前驱节点-1，node直接park，因为前驱节点优先于node获取锁
         */
        return true;
    if (ws > 0) {
        /*
         * ws>0 前驱节点被取消，跳过cancelled前驱节点，直到一个没有被canceled的前驱节
         * 如果执行完循环node.prev=head，则node会尝试获取锁。
         *
         * 当sync队列中有节点后head.waitStatus=SIGNAL
         * Predecessor was cancelled. Skip over predecessor
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        // 前驱节点next指向node
        pred.next = node;
    } else {
        /* ws = -2说明节点在condition，这里不会出现
         * waitStatus must be 0 or PROPAGATE(-3).  Indicate
         * need a signal, but don't park yet.  Caller will 
         * retry to make sure it cannot acquire before park
         * 一般来说每个Node节点初始化waitStatus=0，在加入新节点node后，在下面操作会将前
         */
        // 设置前驱节点状态为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    // 不park线程
    return false;
}
```

        shouldParkAfterFailedAcqure()方法主要是通过node的前驱节点来判断当前节点是否需要被阻塞，需要返回true，否则返回false。

1. 如果node的前驱节点状态=SIGNAL = -1，说明同步队列中node之前还有需要竞争锁的线程，要阻塞当前线程，直接返回true。

2. 如果node的前驱节点状态>0，说明node的前驱节点被取消了，此时会从node开始往前遍历同步队列，找到第一个没有被取消的前驱节点，如果存在被取消的节点，会被移出同步队列。返回false，继续执行for循环重新进入到该方法

3. 如果node的前驱节点状态<=0（节点初始化后状态=0），需要将前驱节点状态修改为-1，说明当前节点之前有需要竞争锁的节点，返回false，继续执行for循环，如果当前节点不是head.next，则继续进入该方法，执行到步骤1会被返回true进行阻塞。

    如果shouldParkAfterFailedAcqure()方法返回true，代表当前线程需要被阻塞，而方法parkAndCheckInterrupt()就是实现线程的阻塞与中断监测功能的：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // park当前线程
    return Thread.interrupted(); // 返回线程中断状态
}
```

### 1.3.3 selfInterrupt()方法

        当前线程自己中断

```java
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

# 二、release&tryRelease

        release()是AQS的模板方法，tryRelease则是一个钩子方法，这里借助ReentrantLock类中的方法进行分析。

## 2.1 tryRelease源码分析

        tryRelease()方法，是用于尝试释放锁（操作state），如果完全释放了锁返回true，如果释放失败，或者没有完全释放返回false。

```java
protected final boolean tryRelease(int releases) {
    // c = 释放锁后state的值
    int c = getState() - releases;
    // 如果当前线程不是工作线程，抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    // free表示可重入锁是否完全释放
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 更行state
    setState(c);
    return free;
}
```

## 2.2 release源码分析

```java
public final boolean release(int arg) {
    // 尝试释放锁，返回值表示是否完全释放锁（存在重入锁情况）,true完全释放
    if (tryRelease(arg)) {
        Node h = head;
        // 头节点不为null
        if (h != null && h.waitStatus != 0)
            // 唤醒head.next
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

1. 调用tryRelease()方法释放锁

2. 如果锁被完全释放了，需要从同步队列中unpark一个node，去竞争锁。被unpark的线程会继续执行aquire()的代码。

### 2.2.1 unparkSuccessor()

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     * 获取head的状态
     */
    int ws = node.waitStatus;
    // 如果node节点为负
    if (ws < 0)
        // 比较并设置节点等待状态，设置为0
        compareAndSetWaitStatus(node, ws, 0);
    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    // 获取node的下一个节点
    Node s = node.next;
    // 如果被唤醒的节点不存在，获取被取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从尾结点开始从后往前开始遍历，防止漏掉为完全入队的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread); // 唤醒阻塞节点
}
```