---
title: 深入了解异步servlet
date: 2018-09-07 13:48:54
toc: true
comment: true
tags:
 - 异步servlet
---

## 一，前言

> 其实为什么想看一下异步Servlet，就是因为在项目中有用到长轮询做变更推送，没有接触过觉得很神奇，结果看了下别人的博客是09年写的，无力吐槽... 只能说自己还是太孤陋寡闻了 。

[https://www.javaworld.com/article/2077995/java-concurrency/java-concurrency-asynchronous-processing-support-in-servlet-3-0.html](https://www.javaworld.com/article/2077995/java-concurrency/java-concurrency-asynchronous-processing-support-in-servlet-3-0.html)

这篇博客从服务器http请求连接的单线程处理单个请求开始，一直讲到异步servlet的出现理由使用以及其他的一些servlet3.0特性。

看了下日期，2009年写的 ... 
里面详细介绍了服务器推送的三种实现方法：
 1. Server streaming: 维持一个客户端与服务器的长连接，服务器需要推送数据的时候发数据即可；
 2. Long polling: 客户端无脑轮询和服务器推送的结合，服务器保持住客户端发来的请求，等到有事件产生的时候才推送数据，客户端收到数据后马上发起下一个请求；
 3. Passive piggyback: 服务器有需要推送的数据，等到下一次客户端的请求到来的时候，将这些数据一起发送回去。

还简单讲了一下异步servlet的实现和作用：
实现：使用额外的线程池处理空闲的连接或者是耗时较长业务逻辑耗时长的连接的读写任务，让占用的服务器主线程可以直接处理下一个到来的客户端请求；
作用：相比thread-per-request模型，有效减轻服务器负担，以及提高了系统的吞吐量。

下面会具体到代码的层面来分析SpringMVC是怎么支持异步servlet的，以及Tomcat是如何处理异步Servlet的，还有了解一下Servlet3.1 针对异步Servlet提出的非阻塞IO是什么东西以及如何实现的。

## 二，异步servlet api

了解一下servlet 3.0 中定义的异步servlet的api 。

### 2.1 AsyncContext
异步servlet提供了AsyncContext这样一个代表异步上下文的类，可以往其中添加异步处理的监听器，和进行请求的分发等等。

使用起来也是比较简单：

``` java
	@Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        AsyncContext asyncContext = request.startAsync();
        asyncContext.addListener(new MyListener());
        process(asyncContext, request);
        logger.info("[servlet] exit service method .");
    }
	
	// xxxx 
	// 在之后开一个线程处理完成之后 调用 asyncContext.complete() 方法就行
	
```
在startAsync() 之后，调用 complete() 方法或者 dispatch() 方法回到处理请求的线程池中，等待线程池中线程处理。

## 三，Spring MVC的异步Servlet

### 3.1 简单使用

文档：[https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-ann-async](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-ann-async)

使用DeferredResult或者Callable作为异步请求的返回结果，都是可以的，需要在web.xml配置中开启异步支持，使用Callable的话需要在Spring配置文件中指定执行异步业务逻辑的线程池。

简单的代码：

``` java
	@GetMapping("/user/{userName}")
    @ResponseBody
    public DeferredResult<User> findUser(
		@PathVariable(name = "userName") String userName) {
		
        DeferredResult<User> result = new DeferredResult<>();
        // request processed by other thread
        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(sleepSecond);
                logger.info("Other Thread set user value ..");
                User user = new User(userName, myAge);
                result.setResult(user);
            } catch (InterruptedException e) {
                logger.error("thread interrupted !", e);
            }
        }).start();
        return result;
    }

    @GetMapping("/callable/user/{userName}")
    @ResponseBody
    public Callable<User> findUserByCallable(
		@PathVariable("userName") String userName) {
        return () -> {
            TimeUnit.SECONDS.sleep(10);
            return new User(userName, myAge);
        };
    }
	
```

### 3.2 DeferredResult

DeferredResult 和 Callable 作为返回结果的区别主要在于异步请求处理过程的不同，官方文档阐述了DeferredResult在spring mvc中是怎么被处理的：

>DeferredResult processing:
>1.Controller returns a DeferredResult and saves it in some in-memory queue or list where it can be accessed.
>2.Spring MVC calls request.startAsync().
>3.Meanwhile the DispatcherServlet and all configured Filter’s exit the request processing thread but the response remains open.
>4.The application sets the DeferredResult from some thread and Spring MVC dispatches the request back to the Servlet container.
>5.The DispatcherServlet is invoked again and processing resumes with the asynchronously produced return value.

靠着之前分析的一点底子，花了点时间总算把DeferredResult在Spring Mvc中的处理流程捋清了。

1，请求进入到**RequestMapping**注解的方法的过程和普通的同步请求没有区别，都是
>DispatcherServlet ->HandlerMapping -> HandlerExecutionChain -> HandlerAdapter -> preHandle -> handle

2，反射调用请求对应的**RequestMapping** 注解方法之后，会得到一个DeferredResult<T>这样的一个返回值，实际上Spring MVC就是在调用完方法之后，对返回值进行判断，如果是DeferredResult说明是异步Servlet，需要做额外的处理：

核心代码在**ServletInvocableHandlerMethod#invokeAndHandle**方法中：

``` java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		// 得到反射调用处理方法的返回值
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		setResponseStatus(webRequest);
		
		// ... 
	
		mavContainer.setRequestHandled(false);
		try {
			// 用返回值处理器来处理返回值(处理像@ResponseBody这样的返回值)
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
			}
			throw ex;
		}
	}
```

后面的代码实际上就是判断一下返回值的class类型是否是DeferredResult class的子类，如果是的话，就会进入异步servlet的处理逻辑，其实就是调用
servlet api的**dispatch** 方法。



