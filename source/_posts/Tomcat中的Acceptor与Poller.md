---
title: Tomcat中的Acceptor与Poller
date: 2018-05-29 13:48:54
toc: true
comment: true
tags:
 - Tomcat
---

## 一，前言

### 1.1 Acceptor 

Tomcat中Acceptor线程主要负责三部分的工作：

 1. 调用**AbstractEndPoint#countUpOrAwaitConnection()** 方法完成对LimitLatch的调用；
 2. 调用**AbstractEndPoint#serverSocketAccept()** 方法完成从服务器Socket接收一个到来的连接， 这个方法调用的ServerSocketChannel在nio初始化的时候被设置为阻塞的，所以说所有的Acceptor线程在执行的时候都会阻塞直到有连接到来。
 3. 接收到新的Socket连接之后，调用**AbstractEndPoint#setSocketOptions(SocketChannel)** 方法对其进行属性的设置，包括对socket属性的设置和将新创建的socket注册到poller上。

简略代码:

``` java
public void run(){
	// 一些endPoint运行状态的判断
	...
	
	endpoint.countUpOrAwaitConnection();
	
	// accept a socket
	U socket = null;
	socket = endpoint.serverSocketAccept();
	
	// configure the socket
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
```

<!--more-->

### 1.2 Poller

看一下Poller内部属性：

![poller attribute][1]

- 一个Poller线程内置了一个Selector，用于注册Acceptor接收的连接。
-  一个**SynchronizedQueue** 事件队列，用于存储Acceptor提供的事件，并且每次Poller运行时候都会将队列中的事件进行处理；
-  volatile boolean 变量：close，指示poller是否被关闭；
-  nextExpiration: long，下一次到期时间，主要用于指示Poller线程每个循环执行完毕后是否需要进行时间等待；
-  wakeupCounter: AtomicLong ，统计当前事件队列的未处理事件的数目，使用方法我觉得非常巧妙，后面细说。
-  keyCount: volatile int ，设置为volatile便于线程外部统计当前连接数。

接下来可以看一下Poller线程运行时候所做的事情：
我所理解的主要做两件事情：

 1. 往内置的selector上面注册channel以及感兴趣的事件
 2. 对selector上已注册的channel，就绪的keys进行处理。

需要注意的是，两件事情的一起处理的话，线程每个循环运行时间可能比较长，如何在两件事情之间取得一个比较合理的平衡呢？后面细说。

## 二，Acceptor线程与Poller线程的交互

其实二者之间的交互核心比较简单，在Acceptor线程接收到新的连接，创建新的Socket之后，会调用**AbstractEndpoint#setSocketOptions(SocketChannel)** 方法进行socket属性的一些设置，在这个方法中将SocketChannel包装成NioChannel对象，然后获取合适的Poller线程，再将这个NioChannel对象包装成PollerEvent对象添加到Poller线程自身的事件队列中，Poller线程每次运行的时候都会对事件队列中的事件进行处理，遍历所有的事件，将事件中的channel和感兴趣的操作注册到自身的Selector上面。

我觉得还是画一个图比较清晰：

![enter description here][2]

emm，大概就是这样的一个处理流程。

## 三，一些细节

其实上面只是从脉络上面梳理了Acceptor和Poller之间的交互，其实有比较多的细节是值得注意的。

 ### 3.1 将连接请求均匀分发到Poller上
 
 Q: 在高并发的情况下，开启的Acceptor线程数目可能非常多，如何保证所有的Acceptor的请求能均匀的落到Poller数组的每一个Poller上呢？
 
 A：解决的办法在于使用一个全局AtomicInteger变量记录poller数组的偏移量，每次分配Poller的时候都将其自增加一，然后取余Poller数组长度即可。
 
 Q：自增数值超过Integer上限怎么办？
 A：自增超过上限**Integer.MAX_VALUE** 其实会得到一个**10000...00000** 共31个0的二进制，也就是**Integer.MIN_VALUE**，对其取绝对值即可。
 
 具体实现：
 
 ![getPoller0][3]

### 3.2 优先处理Poller事件队列中的事件

Poller的设计理念是**事件永远是优先处理**的，（我甚至看到一些书籍说Poller是事件驱动的，我觉得这并不能算事件驱动）。

Q：如果Poller线程每个循环执行花费的时间较多，那么高并发情况会出现一次循环结束后其事件队列已经积累了非常多待处理的事件，那么下一次循环必须完全处理完这些事件，处理事件的期间如果有新的事件到来怎么办？
A：处理事件期间到来的事件还是会加入到队列的尾部，但是并不会在这一个循环中对其进行处理，可以看一下处理事件的代码，只是遍历开始时刻的队列长度的事件，对于后面到来的事件不管了：

![events][4]

Q：高并发下，如果处理事件的时间比较长，并且轮询selector上就绪channel的时间也比较长，那么Poller每一次循环花费的时间岂不是很长？这样的话，就绪的channel等待的时间也非常长，极其影响用户体验，出现负载比较大的情况怎么办？
A：答案在于使用**wakeupCounter** 这样的一个AtomicLong属性。
1，wakeupCounter在每次往事件队列添加事件的时候都会被加1，这样可以统计事件队列中的事件数目，然后在每次处理完一次循环中的事件之后将其置为-1，置为-1之后就进入**selector.select(timeout)** 这样的一个阻塞方法等待IO就绪的channel，直到超时。
2，如何优先处理事件？在阻塞等待超时的过程中，如果事件队列中又有事件到来，**wakeupCounter** 会变为0，此时会调用**selector.wakeup()** 直接唤醒阻塞等待IO就绪的Poller线程加快进入下一个循环。代码实现：

  ![addEvent][5]

  ![run.select][6]
  

3，上面的代码可以看到，将**wakeupCounter** 置为-1后会对旧值进行判断，如果大于0说明Poller的负载有点重，因为事件队列中有事件，为了避免出现负载过重的情况，就直接非阻塞**selector.selectNow()** ，如果之前事件队列为空，就会阻塞select。
4，如果select没有找到就绪的channel怎么办，丧心病狂地Poller线程会再次处理事件队列中的事件，为了优先处理事件。

Q：如果没有事件有没有就绪channel，那岂不是Poller进入一个空的无限循环？
A：在每个循环的末尾会对当前循环是否有事件，是否有就绪channel进行判断，如果都没有说明负载较轻，会进入一个对selector上的所有channel的读写操作是否超时的遍历的判断，如果超时，那么就会往线程池添加一个处理超时Socket的任务。

## 四，后续

对于就绪channel的处理，会将整个处理任务打包成一个**SocketProcessor** 扔到后面的一个线程池进行处理。

后面可以对这方面进行分析。


  [1]: http://ww1.sinaimg.cn/large/006pluSpgy1g0blwpm3gjj30n2097dgy.jpg
  [2]: http://ww1.sinaimg.cn/large/006pluSpgy1g0blx5fqohj30vn0hxac3.jpg "UML时序图"
  [3]: http://ww1.sinaimg.cn/large/006pluSpgy1g0blxp59kwj30rs040wes.jpg
  [4]: http://ww1.sinaimg.cn/large/006pluSpgy1g0blxuhjvgj30rs0is75w.jpg
  [5]: http://ww1.sinaimg.cn/large/006pluSpgy1g0blxzqam0j30rs057t91.jpg
  [6]: http://ww1.sinaimg.cn/large/006pluSpgy1g0bly4kw36j30rs0bqmye.jpg