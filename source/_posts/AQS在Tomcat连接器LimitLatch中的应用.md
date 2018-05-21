---
title: AQS在Tomcat连接器LimitLatch中的应用
date: 2018-05-21 13:48:54
toc: true
comment: true
tags:
 - AQS
 - Tomcat
---

今天重新具体看了一次Tomcat Connector中Acceptor类以及NioEndPoint类的代码实现，感觉还是收获颇多的，在代码里发现了AQS的应用，特意记录一下其使用的场景以及情况。

## LimitLatch

### 作用

LimitLatch 是Tomcat中用来控制Tomcat连接数目的一个阀门，主要起到的作用是当连接请求的数目超过了Tomcat配置的最大连接数时，这时候超出阀门的连接请求到来，阻塞启动的Acceptor线程，这样的话Acceptor就无法调用 **NioEndPoint**的**serverSocketAccept()** 方法进行连接的建立。

显然只有当其他的连接请求处理完成并且socket被关闭之后，Acceptor线程才能继续执行。

如何实现这样的控制效果呢？

<!--more-->

### 实现

#### 设计思想

使用ActomicLong count变量记录当前的连接数目，使用AQS共享模式，共享模式下当连接没有达到上限的时候都可以立刻获取锁成功，马上返回，这样的话实际上只是做了更新连接数目的操作。

当连接数目达到设置的limit变量的时候，AQS就会起作用了，阻塞所有尝试获取共享锁的线程，所有的Acceptor线程都会被阻塞，只有其他连接请求处理完毕，在关闭socket的时候count减一，这样的话才会允许阻塞的和释放的count等量的Acceptor线程进行请求连接的创建。


#### 调用流程

![enter description here][1]

整个代码的执行过程核心调用如上图，可以看到LimitLatch的**countUpOrAwait()** 方法就是起到限制连接作用的核心。

**NioEndpoint#serverSocketAccept()** 方法是获取一个socket连接，**setSocketOptions()** 负责设置socket的相关属性，默认的nio实现是将socket注册到某个poller的selector之上以及设置相关的属性。

#### Acceptor.run()

粗略看一下Acceptor的run方法:

``` java
public void run(){
	...
	// 一些endpoint状态的判断
	
	try{
			// 获取共享锁
		     endpoint.countUpOrAwaitConnection();
			 
			 ...
			 
			 try{
			 	    socket = endpoint.serverSocketAccept();
			 }catch(Exception ioe){
			 		// 获取socker失败，count -1
			 	    endpoint.countDownConnection();
			 }
			 
			 // Configure the socket
                if (endpoint.isRunning() && !endpoint.isPaused()) {
                        // setSocketOptions() will hand the socket off to
                        // an appropriate processor if successful
                        if (!endpoint.setSocketOptions(socket)) {
                            endpoint.closeSocket(socket);
                        }
                } else {
                    endpoint.destroySocket(socket);
                }
	}
}
```

可以看到控制连接的关键在于进行socket获取之前的**endpoint.countUpOrAwaitConnection()** 方法调用。

#### NioEndpoint.countUpOrAwaitConnection()

``` java
protected void countUpOrAwaitConnection() throws InterruptedException {
        if (maxConnections==-1) return;
        LimitLatch latch = connectionLimitLatch;
        if (latch!=null) latch.countUpOrAwait();
    }
```
简单调用**latch.countUpOrAwait()**。

#### LimitLatch.countUpOrAwait()

``` java
public void countUpOrAwait() throws InterruptedException {
        if (log.isDebugEnabled()) {
            log.debug("Counting up["+Thread.currentThread().getName()+"] latch="+getCount());
        }
        sync.acquireSharedInterruptibly(1);
    }
```

核心在于LimitLatch内置的AQS子类**Sync**，以可以响应中断的形式获取共享锁。

#### LimitLatch.Sync

Sync 继承AQS类，覆写了 **AQS.tryAcquireShared(int)** 以及 **AQS.tryReleaseShared(int)** 方法：

``` java
		
		protected int tryAcquireShared(int ignored) {
			// 每当有线程请求获取共享锁的时候，都进行count加一
            long newCount = count.incrementAndGet();
            if (!released && newCount > limit) {
                // 超过限制，减去更新，直接返回-1，让线程阻塞等待
                count.decrementAndGet();
                return -1;
            } else {
				// 返回1 >0 ， 线程会直接返回
                return 1;
            }
        }
		
		@Override
        protected boolean tryReleaseShared(int arg) {
			// count -1 ， 有线程阻塞的话，会触发唤醒。
            count.decrementAndGet();
            return true;
        }
```

内部的Sync类其实逻辑非常简单，有线程请求锁就count +1 ，并且count没有达到limit的时候直接返回1，让线程得以马上继续执行。当count达到limit时，会返回-1阻塞请求的线程。

释放锁的话，就是更新count，让阻塞的线程在进行下一次**tryAcquireShared()** 的时候，返回1，得以执行。

#### LimitLatch.countUpOrAwait()

``` java
public void countUpOrAwait() throws InterruptedException {
        if (log.isDebugEnabled()) {
            log.debug("Counting up["+Thread.currentThread().getName()+"] latch="+getCount());
        }
        sync.acquireSharedInterruptibly(1);
    }
```
对Sync的封装，并且打印日志，调用AQS的**acquireSharedInterruptibly()** 方法进行获取共享锁。

#### LimitLatch.countDown()

``` java
public long countDown() {
        sync.releaseShared(0);
        long result = getCount();
        if (log.isDebugEnabled()) {
            log.debug("Counting down["+Thread.currentThread().getName()+"] latch="+result);
    }
        return result;
    }
```

打印日志释放锁，也是对Sync的封装，对外提供简单易操作的释放锁的接口。

## 结尾

LimitLatch作为进入Tomcat的第一个阀门，利用AQS加上AtomciLong简单地实现了一个阀门控制器，当超过limit限制的连接到来，之后的所有Acceptor线程都会被阻塞无法继续获取socket。
 
  [1]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/21/SpringBeans%20UML.png "SpringBeans UML"