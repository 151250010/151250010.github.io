---
title: 深入解读springmvc--3，剖析RequestMappingHandlerMapping
date: 2017-08-19 13:48:54
toc: true
comment: true
tags: 
 - springmvc源码 
---

> 又到了我最喜欢的啃代码时间。

## 一，前言

>上篇博客写过一个MVCSource项目，把web.xml中的CustomDispatcherServlet换为Spring的DispatcherServlet，运行的话，看server log会发现整个spring mvc的运行轨迹：

![enter description here][1]

1, init DispatcherServlet
2, begin to refresh webapplicationcontext for DispatcherServlet whose namespace is “dispatcherServlet-servlet”
3, load bean definitions from the xml configuration declared by contextConfigLocation in initParam of dispatcherServlet
4, RequestMappingHandlerMapping tries to map and save all (url，handler) defined by @RequestMapping
5, RequestMappingHandlerAdapter tries to look for all exception handler defined by @ControllerAdvice
6, SimpleUrlRequestMapping register a handler DefaultServletHttpRequestHandler for path [\/**]
7, init DispatcherServlet successfully

<!--more-->

可以看到核心是RequestMappingHandlerMapping以及RequestMappingHandlerAdapter，HandlerMapping是处理请求映射的处理器，HandlerAdapter是适配器，负责动态调用方法以及处理参数，静态资源的请求使用的Handler是ResourceHttpRequestHandler对应的HandlerAdapter也是不一样的，但是我们最常用的还是@RequestMapping注解然后调用方法处理请求的方式，这种情况下使用的是HandlerMethod作为Handler，对应的HandlerMapping就是RequestMappingHandlerMapping，接下来我们深入剖析一下RequestMappingHandlerMapping。

## 二，RequestMappingHandlerMapping 的继承关系

如图：

![enter description here][2]

|    关键类或者接口 |  主要实现的功能   |
| --- | --- |
|   Aware接口  |  实现了Aware接口或者各种子接口的类，并且都交由IOC容器进行管理的话，会注入需要的上下文环境或者是 ServletContext   |
|   EmbeddedValueResolverAware 接口	  |  注入一个Spring提供的EmbeddedValueResolver用于进行配置的读取   |
| ServletContextAware接口	    |  获取当前类运行的ServletContext   |
|  ApplicationContextAware接口	   |  获取当前实现类运行的上下文环境   |
|  WebApplicaionObjectSupport抽象类	   |  运行在Web上下文的类的基类，继承该类就可以获取各种上下文以及ServletContext   |
|   Ordered接口	  |   实现该接口的类，可以进行优先级排序，优先级为0就是最高优先级，RequestMappingHandlerMapping优先级就是0，表示是没有意外的话，使用的HandlerMapping都是RequestMappingHandlerMapping  |
|   AbstractHandlerMapping  |  实现了一系列功能的HandlerMapping基类，持有一个Handler以及拦截器列表   |
|   InitializingBean接口	  |   用于初始化bean之后提供一个回调函数进行需要的操作  |
|  HandlerMethod	   |  提供了对一个处理器方法的封装，对外提供一系列的和方法相关的操作   |
|   AbstractHandlerMethodMapping抽象类	  |   对于所有的(url,HandlerMethod)的方法匹配的HandlerMapping提供了基类，并且初始化了所有HandlerMethod与对应的请求路径之间的匹配  |
|RequestMappingInfoHandlerMapping	| 使用RequestMappingInfo来定义HandlerMethod与request之间的映射的基类|

## 三，handlerMethod 初始化 以及 getHandler的流程

initHandlerMethods:

![enter description here][3]

getHandler的处理流程：

![enter description here][4]

具体看下两个流程的实现：

### 3.1，把所有CONTROLLER的METHODS与其对应的HANDLERMAPPINGINFO 记录到内存

1，
AbstractHandlerMethodMapping 实现了InitializingBean 接口，有一个回调函数，IOC容器初始化HandlerMapping实例完成之前，会调用其afterPropertiesSet方法，而afterPropertiesSet方法调用自身的initHandlerMethods:

![enter description here][5]

2，
initHandlerMethods 从默认当前的上下文中获取所有的带有@Controller 以及@RequestMapping注解的实例，并且获取其中的所有的HandlerMethod保存到内存之中。

![enter description here][6]

>isHandler 在RequestMappingHandlerMapping 之中实现：

![enter description here][7]

3，
detectHandlerMethods 获取每个handler类中的所有的提供url映射的(Method,T mapping)的所有映射集合，并且保存到内存之中去：

![enter description here][8]

>getMappingForMethod在RequestMappingHandlerMapping之中实现了，获取类Handler上以及HandlerMethod的RequestMapping的属性构造RequestMappingInfo，并且最后进行合并：

![enter description here][9]

>createRequestMappingInfo ，用获取到的RequestMapping中的属性生产RequestMappingInfo:

![enter description here][10]

![enter description here][11]

>在获取到所有的methods之后，把信息注册到内存之中，以后getHandler实现的，只是从内存map的检索就可以了：

![enter description here][12]

4，
最后initHanlderMethods方法还提供了一个模板方法，供子类在方法与其映射初始化之后进行进一步的操作：

![enter description here][13]

![enter description here][14]

### 3.2，GETHANDLER 获取与请求对应的HANDLEREXECUTIONCHAIN的具体流程：

1，
AbstractHandlerMapping中实现了final方法getHandler：

![enter description here][15]

>主要分三步：getHandlerInternal, getHandlerExecutionChain, getCorsHandlerExecutionChain

2，
getHandlerInternal 方法是一个模板方法，留给子类实现的，在AbstractHandlerMethodMapping中实现了：

![enter description here][16]

>主要是lookupHandlerMethod的实现：

![enter description here][17]

>Match是匹配当前请求的(T mapping, HandlerMethod method)的封装，addMatchingMappings()先往matches列表中增加所有匹配当前请求的Match，接下来根据子类实现的getMappingComparator方法对Match进行排序，直接返回优先级最高的Match之中的HandlerMethod。

看下addMatchMappings的实现：

![enter description here][18]

3，
getHandlerExecutionChain简单的判断一下是不是匹配某个路径的拦截器还是对所有请求路径都适用，然后加入到handlerExecutionChain中：

![enter description here][19]

4，
最后看一下RequestMappingHandlerMapping：

其实RequestMappingHandlerMapping做到事情并不多，主要四部分：

 - 在调用afterPropertiesSet回调函数之前，用AbstractHandlerMapping中的各种属性实例化自身的config：
 
 ![enter description here][20]

- 重写了initHandlerMethods时的两个子类的实现：isHandler以及getMappingForMethod
- 重写了AbstractHandlerMethodMapping之中的initCorsConfiguration跨域配置初始化实现
- 实现了MatchableHandlerMapping的match方法，判断一个请求是否符合标准

## 四，结尾

主要两条主线还是很清晰的。

下一篇实现自定义一个HandlerMapping。

  [1]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504236077006.jpg
  [2]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504236232901.jpg
  [3]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237176346.jpg
  [4]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237187007.jpg
  [5]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237273672.jpg
  [6]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237285595.jpg
  [7]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237300219.jpg
  [8]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237324162.jpg
  [9]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237337448.jpg
  [10]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237351319.jpg
  [11]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237356605.jpg
  [12]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237369696.jpg
  [13]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237397499.jpg
  [14]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237403364.jpg
  [15]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237441417.jpg
  [16]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237906841.jpg
  [17]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237920810.jpg
  [18]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237951393.jpg
  [19]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504237968632.jpg
  [20]: /assets/images/深入解读springmvc--3，剖析RequestMappingHandlerMapping/1504238018228.jpg