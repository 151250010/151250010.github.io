---
title: Tomcat Connector 组件浅析
date: 2018-02-16 13:48:54
toc: true
comment: true
tags:
 - Tomcat
---

## 一，Connector的作用

 官方文档（[http://tomcat.apache.org/tomcat-9.0-doc/architecture/overview.html][1]）：

> Connector负责处理客户端的连接，Tomcat有支持多种协议的Connector，包括Http connector 和 Ajp connector。

Connector主要负责**接收客户端的连接**和对**客户端的请求处理加工**，每个Connector都指定一个端口进行监听，负责对请求报文解析和对响应报文组装。可以理解为Connector组件是请求进入Tomcat的第一道门槛，不同协议的请求可以有不同的门槛进入Tomcat然后被处理。

## 二，Connector 相关部件

![主要部件的作用][1]

<!--more-->

- Http11NioProtocol : Http 协议1.1 版本的抽象，包含接收客户端连接，接收客户端消息报文，报文解析处理，对客户端响应的整个过程。
  - NioEndpoint: Tomcat9 中已经取消了对bio的支持，默认使用nio运行模式，NioEndpoint组件是非阻塞I/O终端的一个抽象，其中包含了很多子组件，其中包括LimitLatch（连接数控制器），Acceptor（套接字接收器），Poller轮询器，Poller池，SocketProcessor（任务定义器）和Executor（任务执行器）。
    - LimitLatch：限制连接数的一个控制器，限制客户端的连接数，nio运行模式下的默认阀门是10000。
    - Acceptor : 主要职责是监听是否有客户端连接进来并且进行连接，生成SocketChannel对象。
    - Poller连接轮询器：主要负责不断轮询事件列表，一旦发现相应的事件则封装成任务定义器SocketProcessor，扔进线程池中执行任务；
     - Poller池：如果只是单个Poller处理所有的客户端事件，那么处理性能可能并没有那么出色，Tomcat使用Poller池，有多个Poller共同处理所有的客户端连接，所有的连接均摊给每个Poller处理。
    - SocketProcessor : 将任务放到线程池处理前所定义的任务执行逻辑。
    - Excutor：用来处理请求任务。excutor线程池的大小就是我们在Connector配置中**maxThread**的值
  - **Http11Processor**: 对Http协议解析并且传递到Engine容器继续处理
- Mapper：客户端请求的路由导航组件，可以通过请求地址找到对应的Servlet。
- CoyoteAdapter: 负责将Connector组件和Engine容器连接起来，把Connector处理得到的Request和Response对象传到Engine容器，调用它的管道。

## 三，Connector 启动过程

以默认的Http11NioProtocol为例，分析启动过程。

### 3.1 Connector初始化与启动

``` java
public Connector(String protocol) {

    if ("HTTP/1.1".equals(protocol) || protocol == null) {
        if (aprConnector) {
            protocolHandlerClassName = "org.apache.coyote.http11.Http11AprProtocol";
        } else {
            // 默认class name
            protocolHandlerClassName = "org.apache.coyote.http11.Http11NioProtocol";
        }
    } 

    ProtocolHandler p = null;
    try {
        // 反射创建Http11NioProtocol对象
        Class<?> clazz = Class.forName(protocolHandlerClassName);
        p = (ProtocolHandler) clazz.getConstructor().newInstance();
    }
}
```
1. Connector 构造函数在启动Tomcat读取**conf/server.xml**之后被调用，用配置文件中的Connector构造实际的Connector。
2. 构造函数中主要反射实例化了Http11NioProtocol对象。
3. 整个Server初始化会带动Connector的初始化，调用Connector中的**initInternal** 方法，首先初始化CoyoteAdapter 并调用setAdapter方法，然后启动Http11NioProtocol对象，并且带动后面一系列组件的初始化。

### 3.2 Http11NioProtocol 初始化和启动

``` java
// 新建NioEndPoint
public Http11NioProtocol() {
    super(new NioEndpoint());

    // 父类的构造函数大致如下
    // this.endpoint = endpoint;
    // ConnectionHandler<S> cHandler = new ConnectionHandler<>(this);
    // setHandler(cHandler);
    // getEndpoint().setHandler(cHandler);
}

// 启动NioEndPoint
public void start() throws Exception {
    endpoint.start();
}
```

1. 新建NioEndpoint，负责接收请求。
2. 新建ConnectionHandler，负责处理请求。
3. 启动NioEndpoint。

### 3.3 NioEndpoint 启动

NioEndpoint 代码实现包含了三个成员组件：

Acceptor，Poller 以及 SocketProcess （处理socket，本质上委托ConnectionHandler处理）。

``` java	
		// NioEndpoint 的初始化工作中会初始化ServerSocket，设置为阻塞模式
	protected void initServerSocket() throws Exception {
        serverSock = ServerSocketChannel.open();
        socketProperties.setProperties(serverSock.socket());
        InetSocketAddress addr = (getAddress()!=null?new InetSocketAddress(getAddress(),getPort()):new InetSocketAddress(getPort()));
        serverSock.socket().bind(addr,getAcceptCount());
        serverSock.configureBlocking(true); //mimic APR behavior
    }
	
public void startInternal() throws Exception {
			...
			// Create worker collection
            if ( getExecutor() == null ) {
                createExecutor();
            }

			// Initialize LimitLatch
            initializeConnectionLatch();

            // Start poller threads
            pollers = new Poller[getPollerThreadCount()];
            for (int i=0; i<pollers.length; i++) {
                pollers[i] = new Poller();
                Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
                pollerThread.setPriority(threadPriority);
                pollerThread.setDaemon(true);
                pollerThread.start();
            }
			
        	// NioEndPoint最核心之处，启动Acceptor
        	startAcceptorThreads();
    }
	
protected final void startAcceptorThreads() {
        int count = getAcceptorThreadCount();
        acceptors = new ArrayList<>(count);

        for (int i = 0; i < count; i++) {
            Acceptor<U> acceptor = new Acceptor<>(this);
            String threadName = getName() + "-Acceptor-" + i;
            acceptor.setThreadName(threadName);
            acceptors.add(acceptor);
            Thread t = new Thread(acceptor, threadName);
            t.setPriority(getAcceptorThreadPriority());
            t.setDaemon(getDaemon());
            t.start();
        }
    }
	
```
NioEndPoint的启动，最主要是创建Acceptor线程池，同时监听新请求。

#### 3.3.1 Acceptor

``` java
public class Acceptor<U> implements Runnable {

    @Override
    public void run() {

        // Loop until we receive a shutdown command
        while (endpoint.isRunning()) {

            try {
                U socket = null;
                try {
                    // 1, 接收新的请求，采用阻塞模式，多个Acceptor线程都会阻塞在此（默认的Acceptor数							目是1）
                    socket = endpoint.serverSocketAccept();
                }

                // 2 设置socket，即调用NioEndPoint的setSocketOptions
                // 将socket添加到Poller池中某个poller
                if (endpoint.isRunning() && !endpoint.isPaused()) {
                    if (!endpoint.setSocketOptions(socket)) {
                        endpoint.closeSocket(socket);
                    }
                } else {
                    endpoint.destroySocket(socket);
                }
            } 
        }
    }
}

public class NioEndpoint {

    protected boolean setSocketOptions(SocketChannel socket) {
        // Process the connection
        try {
            // 3， 将SocketChannel设置为非阻塞模式
            socket.configureBlocking(false);
            // 3， 设置Socket参数值，如Socket发送、接收的缓存大小、心跳检测等
            Socket sock = socket.socket();
            socketProperties.setProperties(sock);

            // NioChannel是SocketChannel的一个的包装类，NioEndPoint维护了一个NioChannel池
            NioChannel channel = nioChannels.pop();
            if (channel == null) {
                // 如果channel为空，则新建一个
            } else {
                channel.setIOChannel(socket);
                channel.reset();
            }

            // 4， 从Poller池取一个poller，将NioChannel交给poller
            getPoller0().register(channel);
        } 
        return true;
    }
}
```

#### 3.3.2 Poller

``` java
public class Poller implements Runnable {

    public void register(final NioChannel socket) {

        // 绑定socket跟poller
        socket.setPoller(this);

        // 获取一个空闲的KeyAttachment对象
        KeyAttachment key = keyCache.poll();   
        final KeyAttachment ka = key!=null?key:new KeyAttachment(socket); 

        // 从Poller的事件对象缓存中取出一个PollerEvent，并用socket初始化事件对象
        PollerEvent r = eventCache.pop();

        // 设置读操作为感兴趣的操作
        ka.interestOps(SelectionKey.OP_READ);
        if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
        else r.reset(socket,ka,OP_REGISTER);

        //加入pollerevent中
        addEvent(r);
    }   
}
```

**register()**方法比较简单，把socket与该poller关联，并为socket注册感兴趣的读操作，包装成PollerEvent，添加到PollerEvent池中。

Poller本身是继承Runnable的可执行线程，如下：

``` java
public class Poller implements Runnable {

    // NIO中的选择器，可以看出每一个Poller都会关联一个Selector  
    protected Selector selector;  
    // 待处理的事件队列，通过register添加
    protected ConcurrentLinkedQueue<Runnable> events = 
            new ConcurrentLinkedQueue<Runnable>(); 

    @Override
    public void run() {

        while (true) {

            boolean hasEvents = false;

            try {
                if (!close) {
                    hasEvents = events();
                    if (wakeupCounter.getAndSet(-1) > 0) {
                        // 1 wakeupCounter指event的数量，即有event
                        keyCount = selector.selectNow();
                    } else {
                        // 1 无event则调用select阻塞
                        keyCount = selector.select(selectorTimeout);
                    }
                    wakeupCounter.set(0);
                }

            } catch (Throwable x) {
                // ...省略
            }

            //either we timed out or we woke up, process events first
            if ( keyCount == 0 ) hasEvents = (hasEvents | events());

            Iterator<SelectionKey> iterator =
                keyCount > 0 ? selector.selectedKeys().iterator() : null;


            // 2 根据向selector中注册的key遍历channel中已经就绪的keys
            while (iterator != null && iterator.hasNext()) {
                SelectionKey sk = iterator.next();
                NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();

                if (attachment == null) {
                    iterator.remove();
                } else {
                    iterator.remove();
                    // 3 处理就绪key
                    processKey(sk, attachment);
                }
            }
        }
    }

    protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
        if (sk.isReadable() || sk.isWritable() ) {
            // 在通道上注销对已经发生事件的关注
            unreg(sk, attachment, sk.readyOps());
            boolean closeSocket = false;

            // 读事件
            if (sk.isReadable()) {
                // 3 具体的通道处理逻辑
                if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                    closeSocket = true;
                }
            }

            // 写事件
            if (!closeSocket && sk.isWritable()) {
                if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                    closeSocket = true;
                }
            }
        }
    }
}

public class NioEndPoint {

    public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {

            // 4 从SocketProcessor池中取出空闲的SocketProcessor，关联socketWrapper
            SocketProcessorBase<S> sc = processorCache.pop();
            if (sc == null) {
                sc = createSocketProcessor(socketWrapper, event);
            } else {
                sc.reset(socketWrapper, event);
            }

            // 4 提交运行SocketProcessor
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                executor.execute(sc);
            } else {
                sc.run();
            }
        } 
        return true;
    }
}
```

1. 调用selector的select()函数，监听就绪事件
2. 根据向selector中注册的key遍历channel中已经就绪的keys，并处理key
3. 处理key对应的channel，调用NioEndPoint的processSocket()
4. 从SocketProcessor池中取出空闲的SocketProcessor，关联socketWrapper，提交运行SocketProcessor

#### 3.3.3 SocketProcessor

``` java
protected class SocketProcessor extends SocketProcessorBase<NioChannel> {
    @Override
    protected void doRun() {

        // ...省略打断代码
        NioChannel socket = socketWrapper.getSocket();
        SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

        if (event == null) {
            // 最核心的是调用了ConnectionHandler的process方法
            state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
        } else {
            state = getHandler().process(socketWrapper, event);
        }
    }
}
```

### 3.4 ConnectionHandler

``` java
protected static class ConnectionHandler<S> implements AbstractEndpoint.Handler<S> {
    @Override
    public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {

        S socket = wrapper.getSocket();

        // 1 获取socket对应的Http11NioProcessor对象，用于http协议的解析
        Processor processor = connections.get(socket);

        // 2 循环解析socket的内容，直到读完
        do {
            state = processor.process(wrapper, status);

            // ...省略超级大一段代码
        } while ( state == SocketState.UPGRADING);
    }
}
```
1. 获取socket对应的Http11Processor对象，用于http协议的解析
2. 循环解析socket的内容，直到读完
3. 后续就是包装成request和response交给CoyoteAdapter

## 四，网络IO读写

上面提到使用Http11Processor对象，解析Http协议，读取和解析Http请求数据，如果是动态内容，会调用用户自定义的Servlet处理类处理并返回结果给浏览器，如果是静态内容，直接返回静态资源数据给浏览器。具体的实现暂不说了，下面可以讨论一下网络IO的读写。

网络IO读写的主要实现类是**NioSelectorPool**，它提供了产生**Selector**对象的功能，通过 **NioSelectorPool.get()** 方法可以获得一个Selector对象。

### 4.1 NioSelectorPool 成员属性

``` java
    // 根据命令行参数-Dorg.apache.tomcat.util.net.NioSelectorShared的设置决定是否在SocketChannel中共享Selector
    // 共享的话，所有的SocketChannel共享一个Selector，
    // 否的话，则每一个SocketChannel使用不同的Selector（开启的Selector对象最多不超过NioSelector.maxSelectors即200）
    protected static final boolean SHARED =
        Boolean.parseBoolean(System.getProperty("org.apache.tomcat.util.net.NioSelectorShared", "true"));

    // 负责处理阻塞模式下的数据读写
    protected NioBlockingSelector blockingSelector;

    // 共享的Selector
    protected volatile Selector SHARED_SELECTOR;

    protected int maxSelectors = 200;
    protected long sharedSelectorTimeout = 30000;
    protected int maxSpareSelectors = -1;
    protected boolean enabled = true;
    
    // 活跃的Selector线程数目
    protected AtomicInteger active = new AtomicInteger(0);
    
    // 备份的Selector数目
    protected AtomicInteger spare = new AtomicInteger(0);
    
    // 缓存Selector
    protected ConcurrentLinkedQueue<Selector> selectors =
            new ConcurrentLinkedQueue<>();
```
### 4.2 Read 和 Write 方法的实现

read方法传入的selector 其实就是get（）方法获得的Selector。
readTimeOut是超时时间。
``` java
public int read(ByteBuffer buf, NioChannel socket, Selector selector, long readTimeout, boolean block) throws IOException {
    //如果是共享Selector和阻塞模式，则使用NioBlockingSelector实现数据读取
    if ( SHARED && block ) {
        return blockingSelector .read(buf,socket,readTimeout);
    }
    SelectionKey key = null;
    int read = 0;
    boolean timedout = false;
    //一开始我们认为是可以读的
    int keycount = 1; 
    //开始时间
    long time = System.currentTimeMillis();
    try {
       //当没有超时，则继续读数据
        while ( (!timedout) ) {
            int cnt = 0;
            if ( keycount > 0 ) {
                cnt = socket.read(buf);
                if (cnt == -1) throw new EOFException();
                read += cnt;
                //如果读取到数据，则继续读
                if (cnt > 0) continue; 
                //如果没有读取到数据，并且不是block模式，则直接break
                if (cnt==0 && (read>0 || (!block) ) ) break; 
            }
            if ( selector != null ) {
                //在NioSelectionPool提供的selector上注册OP_READ事件
                if (key==null) key = socket.getIOChannel().register(selector, SelectionKey.OP_READ);
                else key.interestOps(SelectionKey.OP_READ);
                //调用Selector.select方法
                keycount = selector.select(readTimeout);
            }
            //计算是否超时
            if (readTimeout > 0 && (selector == null || keycount == 0) ) timedout = (System.currentTimeMillis()-time)>=readTimeout;
        } //while
          //如果超时，抛出SocketTimeoutException异常
        if ( timedout ) throw new SocketTimeoutException();
    } finally {
        if (key != null) {
            key.cancel();
            if (selector != null) selector.selectNow();//removes the key from this selector
        }
    }
    return read;
}

``` java
public int write(ByteBuffer buf, NioChannel socket, Selector selector,
                 long writeTimeout, boolean block,MutableInteger lastWrite) throws IOException {
    //如果是共享Selector和阻塞模式，则使用NioBlockingSelector实现写数据
    if ( SHARED && block ) {
        return blockingSelector.write(buf,socket,writeTimeout,lastWrite);
    }
    SelectionKey key = null;
    int written = 0;
    boolean timedout = false;
   //一开始我们认为是可以写的 
   int keycount = 1;
   //记录开始时间
    long time = System.currentTimeMillis(); 
    try {
        while ( (!timedout) && buf.hasRemaining() ) {
            int cnt = 0;
            if ( keycount > 0 ) { 
                cnt = socket.write(buf);
                if (lastWrite!=null) lastWrite.set(cnt);
                if (cnt == -1) throw new EOFException();
               
                written += cnt;
                 //如果写数据成功，重新记录超时开始时间，并继续读
                if (cnt > 0) {
                    time = System. currentTimeMillis(); 
                    continue; 
                }
                //如果写入数据为0，并且是非阻塞模式，则直接退出
                if (cnt==0 && (!block)) break; 
            }
            if ( selector != null ) {
                //在NioSelectorPool的selector注册OP_WRITE事件
                if (key==null) key = socket.getIOChannel().register(selector, SelectionKey.OP_WRITE);
                else key.interestOps(SelectionKey.OP_WRITE);
                keycount = selector.select(writeTimeout);
            }
            //是否超时
            if (writeTimeout > 0 && (selector == null || keycount == 0) ) timedout = (System.currentTimeMillis()-time)>=writeTimeout;
        } //while
          //如果超时，则直接抛出SocketTimeoutException异常
        if ( timedout ) throw new SocketTimeoutException();
    } finally {
         //在返回前，取消SelectionKey， 并将所有的key从selector中删掉
        if (key != null) {
            key.cancel();
            if (selector != null) selector.selectNow();
        }
    }
    return written;
}
```
可以看到，读写数据在非阻塞模式下面并不会监听OP_READ和OP_WRITE事件，如果没有成功就直接返回，读到或者写成功了会继续循环。

但是在阻塞模式下面，如果第一次读或者写不成功，就会在NioPoolSelector提供的Selector上注册读（写）监听位，并循环调用Selector.select(long)方法，等待读写事件的到来，如果读（写）数据成功了就会继续读（写），而循环执行的时间由参数timeOut控制。

读写操作都是第一次尝试直接进行读和写，是一种乐观设计，认为网络大部分情况都是正常的，不会堵塞，如果尝试失败的话，说明网络可能堵塞，那么就注册等待读写就绪事件。

## 五，收获

### 5.1 缓存的使用

NioChannel属于频繁生成与消除的对象，每一个客户端的连接都需要一个NioChannel对其进行处理，对性能的影响非常巨大。

在NioEndpoint中，使用**SynchronizedStack**将上一个客户端使用完之后的NioChannel进行缓存起来，避免对它的回收，这样当新客户端访问到来的时候，只需要替换NioChannel之中的**SocketChannel**对象，NioChannel的其他属性只需要进行重置即可，这样就不需要频繁回收与生成NioChannel对象。

close时的回收：

``` java
private void close(NioChannel socket, SelectionKey key) {
        try {
            if (socket.getPoller().cancelledKey(key) != null) {
                // SocketWrapper (attachment) was removed from the
                // key - recycle the key. This can only happen once
                // per attempted closure so it is used to determine
                // whether or not to return the key to the cache.
                // We do NOT want to do this more than once - see BZ
                // 57340 / 57943.
                if (running && !paused) {
                    if (!nioChannels.push(socket)) {
                        socket.free();
                    }
                }
            }
        } catch (Exception x) {
            log.error("",x);
        }
    }
```

使用缓存中的NioChannel：(NioEndpoint#setSocketOptions)

``` java
			NioChannel channel = nioChannels.pop();
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
                } else {
                    channel = new NioChannel(socket, bufhandler);
                }
            } else {
                channel.setIOChannel(socket);
                channel.reset();
            }
```

对于**PollerEvent**的处理也是这样，PollerEvent是对Poller事件的一个包装类，成员属性有NioChannel和NioSocketWrapper，每次Socket事件处理完之后，将PollerEvent加入一个**SynchronizedStack**将其缓存起来避免GC，下次需要使用的时候只需要替换其中的NioChannel和处理操作标志位即可：

重新使用PollerEvent：

``` java
		public void add(final NioChannel socket, final int interestOps) {
            PollerEvent r = eventCache.pop();
            if ( r==null) r = new PollerEvent(socket,null,interestOps);
            else r.reset(socket,null,interestOps);
            addEvent(r);
            if (close) {
                NioEndpoint.NioSocketWrapper ka = (NioEndpoint.NioSocketWrapper)socket.getAttachment();
                processSocket(ka, SocketEvent.STOP, false);
            }
        }
```



### 5.2 Poller池大小

使用了经典算法：

Math.min(2,Runtime.getRuntime().availableProcessors())，根据Tomcat运行环境决定Poller组件的数量，所以在Tomcat中至少会存在2个Poller，如果Tomcat运行在更多处理器的机器上，则JVM可用处理器个数等于Poller组件的数目：

```java
 	/**
     * Poller thread count.
     */
    private int pollerThreadCount = Math.min(2,Runtime.getRuntime().availableProcessors());
    public void setPollerThreadCount(int pollerThreadCount) { this.pollerThreadCount = pollerThreadCount; }
    public int getPollerThreadCount() { return pollerThreadCount; }
```




[1]: http://ovn5va0pd.bkt.clouddn.com/%E5%B0%8F%E4%B9%A6%E5%8C%A0/13/1520919002812.jpg