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

        之前说过tryAquire()是尝试获取锁，由AQS模板方法调用该方法，以下是ReentrantLock&NonfairSync中的tryAquire()

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

1. 判读锁状态（state值），如果锁空闲，则尝试获取锁，获取成功后将aqs工作线程设置为当前线程，返回。

2. 