---
title: 深入解读springmvc--5，剖析HandlerAdapter
date: 2017-09-04 13:48:54
toc: true
comment: true
tags:
 - springmvc源码
---

## 一，前言

首先看一下**DispatcherServlet**类中的核心方法 **doDispatch** ，罗列一下处理以及分发请求的流程：

1. **getHandler** 遍历自身属性中的 **HandlerMappingList列表** 获取能正确处理该请求的 **HandlerExecutionChain** , **HandlerExecution**包括了处理器以及一个拦截器列表;
2. **getHandlerAdapter** 遍历自身属性中的**HandlerAdapter列表**获取支持该处理器的处理器适配器;
3. 应用 <kbd>1</kbd> 中获取的 **HandlerExecutionChain** 中的拦截器对请求进行预处理;
4. 应用 <kbd>2</kbd> 中获取的 **HandlerAdapter** 调用处理器对请求进行处理，生成模型和视图 **ModelAndView**;
5. 最后就是对 **ModelAndView** 进行渲染。

<!--more-->

之前已经实现了 **getHandler** 的逻辑，这篇博客着重剖析 **HandlerAdapter** ,主要分三部分：
- **Handler,HandlerMapping,HandlerAdapter**三者之间的关系，
- **DispatcherServlet** 中的 **HandlerAdapter** 列表的初始化，
- **HandlerAdapter** 如何调用 **Handler** 正确对请求进行处理。

## 二，Handler,HandlerMapping, HandlerAdapter的关系

### 1，Handler

**Handler** 是一个能够处理请求的对象，SpringMvc中常见的Handler包括之前提过的 **HanlderMethod** ， **ResourceHttpRequestHandler** ， **Controller** 。

三个Handler一般对应的请求处理情况：

| Handler                    | 对应的处理器在代码中的实现                                                             |
|:-------------------------- |:-------------------------------------------------------------------------------------- |
| HandlerMethod              | 对应的是用@RequestMapping注解作用的@Controller类中的方法，是最常用的一个处理器         |
| ResourceHttpRequestHandler | 一般用于处理静态资源的请求，处理方法很简单的把请求的静态文件以流的方式写入到response中 |
| Controller                 | 实现了Controller接口的类，自己实现了一个handlerRequest方法生成ModelAndView             |

### 2，HandlerMapping

**Handler** 是具体处理请求的对象，那么对于一个Http请求对象 **HttpServletRequest** ，SpringMvc是怎么具体找到能够正确处理的 **Handler** 呢，HandlerMapping 就在请求以及处理器中起到了一个桥梁的作用，可以根据每一个请求获取到合适的处理器。

 HandlerMapping其实是一个接口，只需要实现一个 **getHandler** 方法，上一篇我已经尝试过自己实现一个RequestMappingHandlerMapping ，而在 **DispatcherServlet** 中对于每一个请求都是遍历自身维持的  **HandlerMapping列表** 寻找合适的HandlerMapping，调用其  **getHandler** 返回不为空的  **HandlerExecutionChain** 。

 SpringMvc 提供的  **HanlderMapping** 的默认实现包括：  **BeanNameUrlHandlerMapping**   以及  **RequestMappingHandlerMapping** :

| RequestMapping              | 支持处理的Http请求                                                                                                   |
|:--------------------------- |:-------------------------------------------------------------------------------------------------------------------- |
| BeanNameUrlHandlerMapping   | "/bean"的请求会寻找到名字为"/bean"的Spring上下文中的bean，来对请求进行处理，处理请求的是一个注入到Spring上下文中的类 |
| RqeustMappingHandlerMapping | 寻找一个 HandlerMethod 来处理请求，处理请求的是一个@Controller中的带有@RequestMapping注解的方法                      |
| SimpleUrlHandlerMapping     | 可以自己设置urlMap ，也就是url到beanName之间的映射可以自己定义，例如可以自己定义"/test"的请求到"name"的处理器的映射  |

### 3，HandlerAdapter

**HandlerAdapter** 负责完成处理器的动态调用进行请求的处理以及生成  **ModelAndView** ，具体到代码的实现，HandlerAdapter也是一个接口，有三个方法：**suppots** 和 **handle** 和  **getLastModified** ，主要就是前两个方法，**supports** 检测该HandlerAdapter实例是否支持对某一类型的Hanlder的适配，  **handle** 就是用处理器处理请求并且生成模型和视图的过程。

HandlerAdapter定义的三个方法：

![HandlerAdapter的定义](http://ovn5va0pd.bkt.clouddn.com/015779f39fa61108f81d2f8939e84c82.png)

 SpringMvc提供的默认的**HandlerAdapter实现** 包括 ：

| HandlerAdapter                 | 支持的处理器类型                                                             |
|:------------------------------ |:---------------------------------------------------------------------------- |
| HttpRequestHandlerAdapter      | HttpRequestHandler的实例，典型的是对静态资源请求的ResourceHttpRequestHandler |
| SimpleControllerHandlerAdapter | 实现了Controller接口的实例                                                   |
| RequestMappingHandlerAdapter   | 带有@RequestMapping注解的处理器方法                                          |

## 三，DispatcherServlet中HandlerAdapter列表的初始化

DipsatcherServlet中的HandlerAdapter列表的初始化和HandlerMapping列表的初始化都是在  **FrameworkServlet** 中的  **initWebApplicationContext**方法中生成当前的DispatcherServlet的web上下文之后用其中的HandlerMapping，HandlerAdapter等的所有实现，进行DispatcherServlet自身的HandlerMapping列表，HandlerAdapter列表的初始化。

整个初始化的流程：

![DispatcherServlet的初始化](http://ovn5va0pd.bkt.clouddn.com/4f5bd2add250df65193a3b280e2b5f4e.png)

DispatcherServlet自身的 **inintStrategies()** 方法就是用创建的web上下文中的各种SpringMvc组件来初始化  **DispatcherServlet** 自身的各种属性：

![initStrategies](http://ovn5va0pd.bkt.clouddn.com/89a170a3ba1567e0966b018b94daa18a.png)

其中的  **initHandlerAdapters** 的实现就是在web上下文(也可以指定包括web上下文的根上下文)中寻找HandlerAdapter的实现，全部来初始化 **DispatcherServlet**  维持的  **HandlerAdapter列表** ，具体的实现不细说了。

## 四，HandlerAdapter handle方法的具体实现

### 1，静态资源请求

#### 配置SpringMvc组件

对于静态资源请求，一般在Spring配置文件中加上：

![mvc:resource](http://ovn5va0pd.bkt.clouddn.com/6319496991ace0ef4575d88f7801364e.png)

mapping是ant匹配请求路径， location 是对应请求路径的静态资源文件位置。

首先需要清楚的是  **mvc:resource** 标签，Spring会用  **ResourceBeanDefinitionParser** 类来进行处理，而  **ResourceBeanDefinitionParser** 所做的事情就是用配置的  (mapping，location)作为 urlMap属性， 注册生成一个 **SimpleUrlHandlerMapping** ，并且与其相对应的处理器  **ResourceHttpReqeustHandler**  和处理器适配器 **HttpReqeustHandlerAdapter** ，注册到上下文之中去 ：

![registerDefaultComponent](http://ovn5va0pd.bkt.clouddn.com/902167b33178b74d134f73f87391369f.png)

#### 请求处理详细过程

一个静态资源的请求在 **DipsatcherServlet** 的  **doDispatch** 中，会用注册了匹配的请求路径的  **SimpleUrlHandlerMapping**  找到匹配的  **ResourceHttpRequestHandler** ，然后由  **HttpRequestHandlerAdapter** 调用  **ResourceHttpRequestHandler** 的  **handlerRequest** 方法，往response的输出流中写入请求的资源。

![静态资源处理流程](http://ovn5va0pd.bkt.clouddn.com/b89e3d1a27ba3d6248e41a7a155b2a94.png)

### 2，动态请求处理

动态请求最典型也最常用的就是使用  **@RequestMapping** 注解的处理器方法对请求进行处理，配套的处理器是  **HandlerMethod** ，HandlerMapping 使用的是  **RequestMappingHandlerMapping** ，HandlerAdapter 使用的是  **RequestMappingHandlerAdapter** ，现在看下  **RequestMappingHandlerAdapter** 的执行流程：

![ReqeustMappingHandlerAdapter执行流程](http://ovn5va0pd.bkt.clouddn.com/8c85607fae1c52229a660cd24076ec13.png)

核心在于：
- [ ] 参数处理
- [ ] @initBinder 注解的参数转换
- [ ] @SessionAttribute 注解的Session参数的实现
- [ ] @ModelAttribute 注解的方法的调用
- [ ] 反射调用处理器方法
- [ ] 返回值的处理


执行过程有点复杂，都留待接下来的几篇博客进行解析。

## 五，结尾

其实这篇博客写到4.2 HandlerAdapter处理动态请求的时候断了，因为有很多很细节的部分我没有注意到的，打算接下来一篇一篇地把细节之处弄明白，这一篇只能算是简单的介绍一下，接下来继续深入。
