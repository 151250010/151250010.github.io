---
title: AQS 源码--1，简介以及条件阻塞的实现
date: 2018-05-03 13:48:54
toc: true
comment: true
tags:
 - AQS
---

## 一，前言

AbstractQueuedSynchronizer  提供了一个依赖于先进先出的等待队列实现的阻塞锁以及相关同步器（信号量，事件等）的上层框架，简称AQS。AQS实现的核心是维护了一个同步变量state以及一个双向的等待线程链表，依赖于state的CAS更新来完成线程的同步管理。

而使用AQS的方法比较简单， 子类只需要重写AQS中的**tryAcquire(int)** && **tryRelease(int)** 或者 **tryAcquireShared(int)** && **tryReleaseShared(int)** 来定义state对于自己的作用以及在这些方法中完成对于state的修改。

AQS默认支持两种线程同步模式，排他 (exclusive) 以及 共享 (shared) ，两种模式的区别主要在于：

 1. 排他模式下，其他线程的 **tryAcquire()** 请求都不会成功，但是共享模式下，多线程的共同请求有可能成功，资源不足时可能请求不成功。
 2. 还有一个区别就是共享模式下的一个线程请求资源成功的话，会唤醒等待队列中的下一个线程去尝试获取资源。

通常的子类只需要重写 **tryAcquire(int), tryRelease(int)** 去使用排他模式， **tryAcquireShared(int) , tryReleaseShared(int)** 去使用共享模式，一般子类只支持一种模式，但是也有一些例外，例如**ReadWriteLock** 同时支持两种模式。

<!--more-->

## 二，Node

AQS的一个静态内部类，定义了等待队列中的节点。

![enter description here][1]

其中的成员属性包括 ：
- **volatile int waitStatus** ：代表当前节点的等待状态，并且在Node类内部定义了一些特殊值:
	- CANCELLED : 1 ，当前节点由于超时或者被中断而进入取消状态，节点永远不会离开这个状态，被取消的节点的线程不再阻塞。
	- SIGNAL：-1 ， 代表当前节点的后续节点处于阻塞等待中，当前节点释放资源或者取消的时候需要唤醒他的后继节点。
	- CONDITION：-2 ， 代表当前节点处于一个**Condition Queue** 中，需要进行**transfer** 操作才能将当前节点从条件等待队列转换到可执行的同步队列中。
	- PROPAGATE：-3，释放共享资源的时候应该会触发传播，传播唤醒后继的资源等待节点，设置为 PROPAGATE 状态是为了确保传播持续下去。（后面会深入探讨状态的作用）
	- 0 ：一般普通同步等待节点的waitStatus都被设置为0 
- **volatile Node prev** ：指向当前节点的前驱节点，前驱节点可以检查当前节点的状态，前驱节点的设置是在当前节点进入同步队列的时候设置的，在当前节点出队的时候会被设置为null，方便gc起效。
- **volatile Node next** ：指向当前节点释放资源的时候需要唤醒的下一个节点
- **volatile Thread thread**：节点内部的线程
- **Node nextWaiter** ：指向Condition队列中的下一个节点，或者是特殊值SHARED ， 特殊值表示共享模式。

Node提供了一些对应属性的判断和获取方法，一个**isShared()** 判断当前节点是否是在一个共享模式同步队列中，一个**predecessor()** 方法获取当前节点的前驱节点。

## 三，ConditionObject

AQS中的两个内部类之一，Condition接口的实现，AQS的Lock实现的基础，主要核心方法是实现的**signal(), signalAll(),await()** 等Condition接口的方法。简单阐述一下 signal() 以及 await() 的执行过程。

### signal()

signal() 方法：

``` java
		public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
```
**isHeldExclusively()** 判断当前的线程是不是独占资源的线程，如果不是的话，直接抛出异常。

然后调用 **doSignal(Node)** 方法唤醒 condition 队列中的第一个节点。

### doSignal(Node)

``` java
		private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```

代码首先进入do 代码块，将 firstWaiter 指向当前唤醒节点的下一个节点，并且将 first 节点的引用置为null ， 接着进入while ，如果将节点从当前的 condition queue 转换到 可执行的 sync queue 失败并且下一个节点不为空的话，循环将下一个节点进行队列转换操作。

### doSignalAll(Node)

``` java
		private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```

就是将整个链表中的所有节点都进行转换，无所谓转换的成功与失败，转换失败的话是由于Node已经处于**CANCELLED** 状态。

### await()

await() 方法提供了可中断的条件等待，代码：

``` java
		public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
		
		/**
		* 检查是否发生中断，返回 THROW_IE (-1) 代表线程在被唤醒之前就被中断了，REINTERRUPT（1）代表中断发生在 signal 之后，0  代表没有发生中断。
		*/
		private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }
		
		
```
执行的过程：

 1. 如果当前线程被中断，抛出中断异常；
 2. 调用**addConditionWaiter()** 方法往wait queue 添加一个对当前线程封装的新的waiter；
 3. 调用 **fullyRelease(Node)** 释放掉当前线程拥有的 **state**，并且保存方法的返回值**savedState** ， 以便唤醒线程之后重新请求相同数目的资源；
 4. 循环判断当前节点是否被转移到可执行的 sync queue 中，如果不是说明没有被唤醒，继续等待，并且检查是否发生了中断；
 5. 从循环跳出的可能有被唤醒或者是发生了中断，无论哪种情况都需要调用**acquireQueued(Node,int)** 方法（后面细说）在队列中请求与阻塞之前等量的资源，并且将 **interruptMode** 设置为 **REINTERRUPT** 如果中断不是发生在线程被signal之前。
 6. 调用**unlinkCancelledWaiters()** 清理一下condition queue 中已经变为CANCELLED状态的Node。
 7. 最后根据中断类型对中断进行处理，如果是THROW_IE那么抛出中断异常，如果是REINTERRUPT那么调用**selfInterrupt()** 进行自我中断。

 ### addConditionWaiter()
 
``` java
 		private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```
往条件等待队列中加入node的时候，首先判断一下最后的节点是否已经被取消，如果被取消需要对condition queue进行清理操作，然后用当前线程构造一个新的Node，加入到condition queue中。

### fullyRelease(Node)

调用此方法完成当前线程拥有state的完全释放，并且返回释放的state数目：

``` java
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
首先的核心在于调用 **getState()** 方法获取当前线程state数目，并且调用**release(int)** 释放掉全部的state。最后如果释放失败的话，改变node的内部状态为CANCELLED。

### isOnSyncQueue(Node)

``` java
	final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);
    }

```
判断一个节点是否已经在 AQS的 sync queue中，代码判断node的状态以及node的前驱节点是否为null，如果node处于CONDITION状态或者前驱节点为null，说明还是在ConditionObject的condition queue中的。

但是如果node.next 存在，说明node已经被通过CAS操作加入到AQS的 sync queue中了， 直接返回true。

一段很长的注解是阐述如果一个node.prev 非空，并不能一定说明node处在AQS的sync queue中，需要从尾开始遍历整个链表。为什么不能说明node在队列中呢？解释是说CAS操作可能存在失败的情况，具体应该怎么理解后面细说。

### ArrayBlockingQueue中Condition的使用

在ArrayBlockingQueue中，维持了ReentrantLock中的两个Condition ： 

![two conditions][2]

可以看一下往队列尾部添加元素的时候对于Condition的使用：

![arrayBlockingQueue#put(E)][3]

在多线程环境下，如果多个put请求同时到来，都会进入到`lock.lockInterruptibly()`方法，多个线程去尝试获取锁，只有获取锁成功的线程会进入到`try ... finally ...` 代码块之中（我给这个线程命个名叫T1），结合**ReentrantLock**的源码来分析的话，其实这个时候获取到锁的线程是**AbstractQueueSynchonizer**中维持的双向链表的head节点，接着代码进入一个while循环，当底层数组已经满了的时候就会一直调用Condition **notFull**的await()方法，也就是我们上面分析的await()方法，用当前线程构造一个新的Node节点，加入到condition queue中，释放掉当前线程使**lock**方法获取的**AQS中的state**资源，并且从AQS的**sync queue**中移除，然后调用**LockSupport.park(this)** 让当前线程（T1）阻塞。

而释放掉的**AQS中的state** 会唤醒其他的尝试获取锁的线程，其他的获取到锁的线程也会进入到**try...finally...** 代码块，如果ArrayBlockingQueue中的底层数组仍然是满状态的话，那么第二个执行的线程（命名T2）也会重复第一个线程的执行流程，加入到condition queue ， 释放资源和阻塞自身。

Q ：那么在Condition中是否存在多线程并发问题呢？会不会有两个线程同时addConditionWaiter()呢？
A：不会，只有成功获取**ReentrantLock** 锁的线程才会继续执行，而Condition使用的是AQS的独占锁，只有拥有独占锁的线程才能进入到 **await()** 或者**signal()** 等，所以不存在线程安全问题。

那么阻塞在等待**notFull** condition上的线程唤醒时的执行流程是怎么样的？

看一下ArrayBlockingQueue中的**take()** 实现:

![take()][4]

假如有个线程T3在T1，T2阻塞的时候，调用了**take()** 方法，和put的过程比较类似，获取lock ， 如果底层数组没有数据的话，循环调用**condition.await()** ， 直到数组已经有数据了，进入dequeue（）方法：

![dequeue][5]

dequeue的实现我们需要关心的是**notFull.signal()** 这样的一个调用，调用**notFull.signal()** 的时候执行的就是上面分析过的**ConditionObject#signal()** 方法：

signal方法执行的第一步是判断调用signal方法的线程是否拥有独占锁，只有拥有独占锁的线程才有资格去唤醒等待的线程，而signal方法做的事情比较简单，就是唤醒**ConditionObject** 中维持的 **condition queue** 中的第一个没有被中断或者被取消的等待节点（我们的condition queue中的第一个等待节点其实就是T1），然后将这个等待节点转移到有执行可能性的AQS的**sync queue** 中，执行可能性怎么理解呢？

上面说过，AQS维持的**sync queue**是一个双向链表，被唤醒的节点会加入到**sync queue**的尾部，只有他前面的有效等待节点都已经被唤醒之后，才会轮到这个被唤醒的节点成功执行。

回到**dequeue()** 代码的执行，T3调用notFull.signal()完成之后，会返回到**take()** 函数，执行**finally** 代码块中的**unlock()** 操作，释放掉当前线程(T3)的锁，其实这样做导致的结果是唤醒AQS中**sync queue** 中的下一个节点执行。

这个时候因为没有其他的线程请求获取**ReentrantLock** 上锁，所以AQS中的**sync queue** 的第一个节点就是刚刚转移的 T1 ， 如果在T3 执行的过程中有线程T4请求锁，那么AQS的队列第一个节点应该就是T4 。

T1 被唤醒，我们可以回到上面看一下await() 代码：

![await()][6]

当T1被唤醒之后，从`while(!isOnSyncQueue(node))` 跳出，尝试获取等量的被阻塞之前的T1所拥有的资源（state），恢复之前的状态，这个过程其实也有可能发生阻塞，可能资源数目不足，**acquireQueued(Node,int)** 的实现下次说。

获取资源成功之后，T1 就会从await() 方法返回，回到**ArrayBlockingQueue#put()** 方法之中，执行**enqueue(E)** 方法往底层数组添加元素，然后执行finally中的**lock.unlock()** 方法，这个方法其实最后也是对AQS中的**state** 进行修改，释放当前线程之前占有的资源(state) ， 并且唤醒AQS sync queue 中的下一个节点执行 。

## 四，LockSupport

在ConditionObject 中，主要使用了**LockSupport.park(Object)** 来阻塞当前线程，并且在AQS的源码中，唤醒下一个节点使用的是**LockSupport.unpark(Thread)** 。 

上面阐述condition执行流程，也没有说park() 以及 unpark() 的使用或者原理。

简单看了一下LockSupport 类上面的注解：
>LockSupport 是用来创建锁和其他同步类的基本原语。

>LockSupport 类以及每个使用它的线程与一个permit相关联，如果permit可用，并且在进程中消耗掉，那么对于**park** 的调用会立即返回，否则可能阻塞，对于**unpark** 的调用将会使还不可用的permit变得可用。（但是permit并不允许累加，最多只能存在一个permit）。

>park 和 unpark 提供了阻塞和解除阻塞线程的有效方法，他们两者之间的竞争会保持活性，不会导致废弃的Thread.suspend 和 Thread.resume 两者存在的死锁的问题，此外，如果调用者线程被中断，或者是使用了parkNanos等有超时限制的方法时候，park会直接返回，因此通常必须在重新检查返回条件的循环方法里面调用此方法，从这种意义上说，park是"忙碌等待"的一种优化，它不会浪费这么多的时间进行自旋，但是必须将它与unpark配对使用才高效。

> 三种形式的park还各自支持一个Blocker对象参数，此对象在线程阻塞的时候被记录，以允许监视工具和诊断工具确定线程受阻塞的原因（这样的话工具可以使用的方法是getBlocker(java.lang.Thread)）访问Blocker ，建议使用带有Blocker参数的park方法。

### unpark(Thread)

- 如果permit不可用的话，使permit可用，也就是park()方法立即返回；
- 如果permit可用的话，参数线程的下一次park()方法调用将保证不阻塞；
- 如果线程还没有启动的话，这个方法不保证执行的结果。

### park(Object)

阻塞当前线程，直到发生了下面三种情况就会马上返回：

 1. unpark() 方法被调用了
 2. 其他的线程中断了当前线程
 3. 方法没有理由地突然返回了

### 结合ConditionObject#await() 场景

在ConditionObject#await() 方法中，使用了 **LockSupport.park(this)** 来阻塞当前线程T1，这样的话，只有将**condition queue** 的T1Node节点转移到AQS的**sync queue** 中，并且被他的前驱节点唤醒的时候，（前驱节点唤醒T1 Node 发生在**AQS#release()** 方法中，调用LockSupport.unpark(s.thread)） 方法：

``` java
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
这样的话被阻塞的T1 线程就可以被唤醒，从while循环中跳出。

## 五，总结

整篇主要分析了一下介绍了一下AQS ， 以及AQS内部类**Node** 和 **ConditionObject**， 着重从源码分析了**Condition** 阻塞的实现。



  [1]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/2/1525234698451.jpg
  [2]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/3/1525332106032.jpg
  [3]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/3/1525332257787.jpg
  [4]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/3/1525335803415.jpg
  [5]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/3/1525336001850.jpg
  [6]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/3/1525338197743.jpg