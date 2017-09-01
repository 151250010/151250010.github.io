---
title: 深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet
date: 2017-08-16 13:48:54
toc: true
comment: true
tags: 
 - springmvc源码 
 - custom spring mvc
---

## 一，一个Http请求进入SpringMVC的完整流程：

![enter description here][1]

<!--more-->

首先看下动态请求的处理：（静态资源请求的处理其实比较简单，只是以流的方式把静态资源写入response中而已）

在Servlet容器中，一个请求的完整流程是：

1.Servlet容器调用Servlet.service()
2.HttpServlet.service()分发get,post等
3.FrameworkServlet.doGet()
4.final FrameworkServlet.processRequest()
5.DispatcherServlet.doService()
6.DispatcherServlet.doDispatch()
7.根据请求的url路径找到可以匹配的HandlerMapping，调用其getHandler得到HandlerExecutionChain，HandlerExecution具体包含了实际的handler处理逻辑以及一组提供额外功能拦截器

> 看下doDispatch中的对getHandler的调用以及getHandler的实现：

![enter description here][2]

![enter description here][3]

*DispatcherServlet初始化完成之后，DispatcherServlet的所有的mvc元素都已经被赋值，所以直接遍历this.handlerMappings调用getHandler，handler不为空直接返回。*

**getHandler的具体逻辑实现以及HandlerMapping中的对的初始化设置，先留下来下一篇继续深入探索。**

8.用HandlerExecution中的拦截器对请求进行预处理，必须要所有的拦截器的preHandler都执行成功才会继续，否则直接返回，请求处理完毕。

> doDispatch中的调用HandlerAdapter以及预处理请求部分

![enter description here][4]

![enter description here][5]

9.预处理成功之后，根据找到的HandlerExecutionChain的handler找到支持该handler的HandlerAdapter，调用HandlerAdapter.handle(request,response,handler)

10.找到一个HandlerAdapter的具体实现（以RequestMapping注解的RequestMappingHandlerAdapter为例）,RequestMappingHandlerAdapter.handleInternal(request,response,method)，用反射调用具体的处理方法，并且生成ModelAndView

> 调用handle方法直接生成ModelAndView:

![enter description here][6]

>RequestMappingHandlerAdapter的实现：

![enter description here][7]

**继续深入的话就是利用反射工具反射调用处理方法，以及ModelAndView的生成。先留着，下下篇继续深入。**

11，在DispatcherServlet中的doDispatch方法获得返回的ModelAndView，最后进行模型和视图的渲染

**视图的渲染也留着下下篇博客继续深入**

## 二，订制自己的DispatcherServlet

啃书啃了半天，来找点成就感吧，准备实现一个侏儒版的DispatcherServlet，把每个部分的功能都砍一遍，自己依托已有的FrameworkServlet写个CustomDispatcherServlet。

分两步吧：
1，先用xml配置一个最简单的SpringMVC项目，并且能够在Tomcat中运行起来；
2，在已有的web.xml中把DispatcherServlet改为我们自己写的CustomDispatcherServlet，再次部署项目到Tomcat中，也依然可以运行。

### 1，配置一个最简单的SPRINGMVC项目，看下我的目录结构：

![enter description here][8]

在web.xml中配置DispatcherServlet和必要的启动参数，还有其映射路径：

![enter description here][9]

spring-test.xml中开启component-scan扫描controller相应包，还有配置一个InternalResourceViewResolver视图解析器：

![enter description here][10]

resource包下写个Hello的controller类：

![enter description here][11]

index.jsp 简单的一个跳转链接：

![enter description here][12]

Hello.jsp 就是say hello而已：

![enter description here][13]

然后配置一下Tomcat，启动项目访问，点击SayHello链接可以进行页面的跳转说明已经成功了！

### 2，把DISPATCHERSERVLET改为自己手写的CUSTOMDISPATCHERSERVLET

接下来就到了重量级的手写CustomDispatcherServlet了，回顾一下DispatcherServlet的主要职责：
- 自己维持mvc的HandlerMapping,HandlerAdapters等元素，对其进行初始化
- doDispatch分发各类的请求

我们主要实现这两部分就ok了，我们的CustomDispatcherServlet继承FrameworkServlet，只有必须的三个成员属性：
**HandlerMapping,HanlderAdatper,ViewResolver**

![enter description here][14]

1，实现onRefresh方法，从FrameworkServlet已经初始化成功的web上下文或者整个应用的根上下文中寻找三个成员属性的实例，对其进行初始化：

![enter description here][15]

initHandlerMappings实现：

![enter description here][16]

initHandlerAdapters和initHandlerMapping的实现差不多，intViewResolver是直接获取配置的InternalResourceViewResolver:

![enter description here][17]

2，重写doService方法，完成对Http请求的处理：

doService设置了必要的参数，后面的视图的渲染需要用到web上下文，所以需要把webapplicationcontext设置到请求域中，但是name不能随便命名，必须是DispatcherServlet.CONTEXT，这个留到后面再看下具体实现，以及将分发任务委托给doDispatch。

![enter description here][18]

doDispatch ：对请求获取其对应的Handler，然后生存ModelAndView，最后进行视图的渲染即可：

![enter description here][19]

getHandlerExecution 就是简单的遍历所有的HandlerMapping，依次执行getHandler方法，若不为空就直接用：

![enter description here][20]

getHandlerAdapter 也是遍历所有的HandlerAdapters，依次判断是否支持该请求对应的handler，支持的话直接返回：

![enter description here][21]

最后是视图的渲染render ：

![enter description here][22]

写到这里，大功告成。

马上用自己的DispatcherServlet来处理请求，把web.xml中的DispatcherServlet去掉，换上自己的：

![enter description here][23]

然后在Tomcat中运行，点击链接SayHello:

![enter description here][24]

成功跳转！

![enter description here][25]

哇，成就感满满的，手写一个SpringMVC框架指日可待了。

## 三，结尾

源码：[https://github.com/151250010/TestWeb][26]

三个遗留问题下篇博客或下下篇解决：

- HanlderMapping 的 键值对是怎么设置的，以及getHandler又是怎么获取到HandlerExecutionChain的
- HandlerAdapter.handle() 生成ModelAndView 到底怎么实现的
- 视图的渲染具体怎么实现的

  [1]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504202988871.jpg
  [2]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203048990.jpg
  [3]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203071618.jpg
  [4]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203158918.jpg
  [5]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203170881.jpg
  [6]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203193591.jpg
  [7]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203214890.jpg
  [8]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203297363.jpg
  [9]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203307781.jpg
  [10]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203318210.jpg
  [11]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203331089.jpg
  [12]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203346929.jpg
  [13]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504203358779.jpg
  [14]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235460250.jpg
  [15]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235471339.jpg
  [16]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235484938.jpg
  [17]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235496771.jpg
  [18]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235514808.jpg
  [19]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235533222.jpg
  [20]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235549929.jpg
  [21]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235564616.jpg
  [22]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235578030.jpg
  [23]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235592704.jpg
  [24]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235602893.jpg
  [25]: /assets/images/深入解读springmvc--2，处理Http请求以及自定义一个DispatcherServlet/1504235614150.jpg
  [26]: https://github.com/151250010/TestWeb