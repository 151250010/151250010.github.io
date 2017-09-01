---
title: 深入解读springmvc--4，自定义一个RequestMappingHandlerMapping
date: 2017-08-23 13:48:54
toc: true
comment: true
tags: 
 - custom spring mvc
---

## 一，前言

>又到了找成就感的深度定制SpringMVC组件的时间，这次我想依托已有的IOC容器，完全自己实现一个发现HandlerMethod以及能够正确处理请求的简化版RequestMappingHandlerMapping，简单处理路径以及请求方法的匹配，对于参数，请求头，跨域请求等先不处理。

其实RequestMappingHandlerMapping主要有两个功能需要实现：

- initHandlerMethods() 发现所有的RequestMapping处理方法，以及将他们保存进入内存等待检索
- getHandler() 检索内存中的处理方法，找到Http请求对应的最佳处理方法，并且可以匹配ant风格的路径检索，还有就是拦截器的处理。

<!--more-->

## 二，必要的几个类

**CustomRequestMappingInfo，PatternRequestCondition，RequestMethodRequestCondition，以及一个工具类RequestPathUtil。**

### 1，ReqeustPathUtil 提供了获取请求在web应用中的路径的方法

首先要搞清楚getContextPath,getServletPath,getUri返回值的意义，举个例子说明(contextPath，servletPath，requestUri)：

![enter description here][1]

整个类提供了一个lookupRequestPath方法，获取请求相对于web应用或者Servlet的路径

![enter description here][2]

![enter description here][3]

使用Tomcat作为ServletContainer的话，就是简单的截取字符串就可以了，但是Jetty的实现有点不同，getContextPath会出现一些特殊的符号，需要进行处理。我只是考虑了使用Tomcat的情况。

### 2，RequestCondition是SpringMVC自己定义的一个请求条件接口

需要实现三个方法：**combine,getMatchingCondition,compareTo**，三个方法顾名思义就是条件合并，获取匹配的条件，条件比较。

### 3， PatternRequestCondition

通过继承AbstractRequestCondition间接实现了SpringMVC定义的请求条件接口RequestCondition，代表的是路径的匹配条件。PatternRequestCondition的属性：

![enter description here][4]

patterns其实就是@RequestMapping注解上面的path参数的值。

最核心的应该是路径匹配的实现：

![enter description here][5]

![enter description here][6]

![enter description here][7]

Ant路径匹配规则的实现，我是直接用了Spring自己实现的，实在是有点复杂，如果需要的话有时间再深入看下AntPathMatcher的实现。

### 4，RequestMethodRequestCondition

RequestMethodRequestCondition是请求方法的匹配条件，属性是@RequestMapping注解的method的值，实现比较简单。

### 5，CustomRequestMappingInfo

CustomRequestMappingInfo组合了PatternRequestCondition和RequestMethodRequestCondition，代表的是一个请求方法的各种条件的组合，我只是简单的组合了路径匹配以及请求方法匹配。用@RequestMapping注解上的path，method的值来构建一个请求方法特有的CustomRequestMappingInfo，每个请求处理方法都会创建一个。

![enter description here][8]

### 6，使用Builder生成mappingInfo的实例

这样写起来代码看着比较舒服，所以CustomRequestMappingInfo写了个静态内部类Builder：

![enter description here][9]

CustomReuqestMappingInfo写个静态方法：

![enter description here][10]

生成CustomRequestMappingInfo就比较简单了，代码简洁明了，看着很舒服：

![enter description here][11]

## 三，处理方法的注册

定义一个实现了**HandlerMapping,InitializingBean,ServletContextAware,ApplicationContextAware,Ordered**
五个接口的**CustomRequestMappingHandlerMapping**类，其中Aware之类的接口是为了从IOC容器中获取相应的(ApplicationContext或者ServletContext)实例，InitalizingBean接口实现是在Bean初始化完成的最后一步提供一个回调函数afterPropertiesSet()，进行所有的HandlerMethod的注册，Ordered接口是配置我们自定义的HandlerMapping的优先级（我设置的是最高优先级order=0）：

![enter description here][12]

### 1，看下CustomRequestMappingHandlerMapping类的属性：

- **SCOPED_TARGET_PREFIX**是特定作用域的Bean的前缀，一般排除掉这样的Bean
- **WebApplicationContext**和**ServletContext**是需要用的web上下文，以及Servlet上下文
- **mappingLookup**是一个处理器方法的所有条件（例如路径，请求方法等）和处理器方法的映射
- **urlLookup**是一个请求url和处理器方法条件之间的映射，其中一个url可以对应多个处理器方法条件，因为可能有两个处理器方法可以处理同一个路径的请求，这样的话我只是简单的选择第一个。

![enter description here][13]

进行处理器方法的注册：

![enter description here][14]

![enter description here][15]

- 获取所有的bean然后判断是不是处理器**isHandler()**，是的话注册一个处理器中的所有的处理器方法**detectHandlerMethods()**。

![enter description here][16]

- isHandler的简单判断一下是不是有**@Controller和@RequestMapping**注解：

![enter description here][17]

- detectHandlerMethods首先获取当前handler的所有方法以及其对应的请求内容的映射**selectMethods()**，然后进行注册**registerMappings()**：

![enter description here][18]

- selectMethods参数有一个Method转化为CustomReqeustMappingInfo的函数性接口，为了代码简洁明了的。调用的时候写了个**getMappingForMethod**方法：

![enter description here][19]

- getMappingForMethod 对于类以及方法的RequestMapping进行了结合然后生成CustomRequestMappingInfo。

![enter description here][20]

- 最后的注册把直接路径就是RequestMapping注解上的path属性没有使用ant通配符的进行直接路径注册，加速以后的检索，还有的一个是把整个处理器方法的请求条件和处理器方法注册。

到此，处理器方法的注册就已经全部完成了。

## 四，根据请求获取相应的处理方法

直接实现HandlerMapping接口的getHandler方法：

![enter description here][21]

三个过程：

### 1，获取请求内路径：简单调用RequestPathUtil的lookupRequestPath方法。

### 2，用请求去查找合适的处理器方法：

![enter description here][22]

如果有直接路径匹配，先去寻找直接路径匹配的处理器方法，没有找到的话，只好遍历所有的处理器方法请求条件，一个个找处理器方法。

![enter description here][23]

但是哈希map是使用**key对象的HashCode**作为寻找value的依据的，所以PatternRequestCondition，RequestMethodRequestCondition，CustomRequestMappingInfo都需要重写**hashCode**方法，PatternRequestCondition以及RequestMethodRequestCondition都是继承AbstractReqeustCondition的，父类已经根据实现的getContent()方法重写了hashCode方法，我们需要做的是简单重写**CustomRequestMappingInfo**的hashCode():

![enter description here][24]

### 3，最后找到处理器方法，不管拦截器直接生成HandlerExecutionChain 返回了。

到此，根据请求查找处理器方法也已经实现了。

## 五，看下成果

把我们的CustomRequestMappingHandlerMapping注入到上下文中，然后运行web项目即可。

![enter description here][25]

点击SayHello：

![enter description here][26]

发现跳转成功：

![enter description here][27]

## 六，结尾

其实mapping注册时候，需要考虑线程安全问题，SpringMVC源码使用用读写锁上锁实现线程安全的，但是阉割版也就这样了。配置之前的CustomDispatcherServlet，成就感还是满满的。

但是啃源码的工作还是极其枯燥，我想着下一篇是看SpringMVC的视图渲染还是写一些成熟工具例如Docker，Jenkins的深度使用，也可以看下Spring之外的框架，例如JFinal，可能会先搞点源码之外的东西来激起兴趣然后再回头看源码了。

源码地址：

[https://github.com/151250010/TestWeb][28]


  [1]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504240914973.jpg
  [2]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504240927633.jpg
  [3]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504240933707.jpg
  [4]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241022278.jpg
  [5]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241037493.jpg
  [6]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241043487.jpg
  [7]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241049518.jpg
  [8]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241115689.jpg
  [9]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241148659.jpg
  [10]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241160988.jpg
  [11]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241174827.jpg
  [12]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241272430.jpg
  [13]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241321770.jpg
  [14]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241333313.jpg
  [15]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241338839.jpg
  [16]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241358123.jpg
  [17]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241373692.jpg
  [18]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241405277.jpg
  [19]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241448306.jpg
  [20]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241467932.jpg
  [21]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241622443.jpg
  [22]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241655915.jpg
  [23]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241666234.jpg
  [24]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241754529.jpg
  [25]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241787480.jpg
  [26]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241796216.jpg
  [27]: /assets/images/深入解读springmvc--4，自定义一个RequestMappingHandlerMapping/1504241806403.jpg
  [28]: https://github.com/151250010/TestWeb