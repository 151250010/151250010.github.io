---
title: 深入解读springmvc--6，@SessionAttribute的工作原理
date: 2017-09-07 13:48:54
toc: true
comment: true
tags:
 - springmvc源码
---

## 一，参考博客

这篇博客参考了以下的博客内容：

[Understanding Spring MVC Model and Session Attributes](https://www.intertech.com/Blog/understanding-spring-mvc-model-and-session-attributes/ "理解SpringMvc模型以及Session属性")

如何区分Spring Model 以及 Session Attribute ，博客作者自己试了 @ModelAttribute 和 @SessionAttributes 的例子，阐明两个注解在处理器方法调用之后才将ModelMap 之中的数据保存进 Request 以及 Session 之中，
我写这篇博客会从源码层面验证 @SessionAttributes 的实现。


## 二，引言

@SessionAttributes 和 @SessionAttribute 是SpringMVC提供的两个注解，用于在请求之间保存模型的属性，以及获取一次会话之间的某个属性值。

### 1，@SessionAttributes

Spring文档之中对于 @SessionAttributes 使用的说明：

> Using @SessionAttributes to store model attributes in the HTTP session between requests
> The type-level @SessionAttributes annotation declares session attributes used by a specific handler. This will typically list the names of model attributes or types of model attributes which should be transparently stored in the session or some conversational storage, serving as form-backing beans between subsequent requests.

简单翻译一下就是： 为了简便地在几次请求之间传递模型数据，SpringMvc 提供了 @SessionAttributes 注解，用于暂时地存放控制器工作信息。但是对于需要添加或者删除 Session 的属性，需要往控制器方法注入 **WebRequest** 或者 **HttpSession** 。

### 2，@SessionAttribute

@SessionAttribute 注解是用于获取已经存在的会话的某些属性，作用于处理器方法的参数之上，填充参数值。

## 三，@SessionAttributes 整体分析

### 1，相关类作用解析

`SessionAttributeHandler`

SessionAttributeHandler 提供了 对 @SessionAttributes 注解标志的控制器的 Session 属性的控制，依赖于 **SessionAttributeStore** 进行  **@SessionAttributes** 指出的Session属性的保存。

@SessionAttributes 指出的属性的执行流程其实很简单，对于每个使用 @SessionAttributes 标志的Controller，在每次往其模型填充属性的时候，都会进行属性的判断，如果和 @SessionAttributes 的names 重名的话，就将数据保存到  **HttpSession** 之中，直到控制器调用 **SessionStatus.setComplete()** 方法，请求完成的最后时刻才会将数据清除。

 **SessionAttributeHandler** 的核心是在 SessionAttributeStore 之外进行了一层的封装，内置一个  **Set<String>**  支持  **记住** 已发现的Session中的属性名称，来支持 **retrieveAttributes** 方法和  **cleanupAttribute**方法的实现。

 `SessionAttributeStore`

 SessionAttributeStore 是一个接口，定义了三个方法：

| MethodName        | Function                    |
|:----------------- |:--------------------------- |
| storeAttribute    | 保存Sessionm属性            |
| retrieveAttribute | 根据AttributeName获取属性值 |
| cleanupAttribute  | 移除具体的属性              |

SpringMVC 只提供了一个默认的实现  **DefaultSessionAttributeStore** :

都是简单调用 WebRequest 实现的  **setAttribute** ，  **getAttribute** ，  **removeAttribute** 方法。

`RequestAttribute`

对一个请求的抽象封装，可以存储和移除Request和Session之中的Attribute。提供了一个统一的属性访问接口。

`WebRequest`

WebRequest 是一个简单的接口，继承了RequestAttribute 接口，可以看成是对 Http请求的一个封装，提供了访问请求属性的很多接口。

在  **RequestMappingHandlerAdapter** 中实例化了  **ServletWebRequest** 作为WebRequest的实现：

![ServletWebRequest](http://photos.zengxihao.xyz/72684809371550e17b659a0081c6a84c.png)

`ModelFactory`

在处理器方法被调用之前进行模型的初始化以及方法调用之后的模型更新。

主要负责处理：
- 处理器方法调用之前，如果有 @ModelAttribute 注解的方法，先进行调用并且将结果加入到控制器的模型之中
- @ModelAttribute注解的方法的调用
- 确保@SessionAttribute标志的属性一定在Session属性之中存在，不在的话抛出异常

### 2，对 @SessionAttributes的处理

**第一步**

简单建立一个@SessionAttributes的使用例子，将userName以及password作为 @SessionAttributes 的names属性：

![SessionAttributes 使用](http://photos.zengxihao.xyz/8d620e003198ab7bf206ede278d20c8b.png)

可以用ModelMap或者@ModelAttribute注解往Model之中填充必需的 userName 以及 password 属性。我用了  **ModelMap** :

![init方法](http://photos.zengxihao.xyz/63dbb0d626b3d0059e00b3b82e5892e5.png)

这样在完成init请求之后，Session之中就应该存在userName,password了。

**第二步**

查看RequestMappingHandlerAdapter的源码，核心方法 **invokeHandlerMethod** 大概如下：

``` java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

        try{
          ...

          ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

          ... // 进行必要的属性的初始化

          modelFactory.initModel(webRequest, mavContainer, invocableMethod); // 对模型数据进行初始化

          ... // 一些异步操作

          invocableMethod.invokeAndHandle(webRequest, mavContainer); // 真正反射调用处理器方法

          return getModelAndView(mavContainer, modelFactory, webRequest); // 生成模型和视图
        }
        finally {
          webRequest.requestCompleted(); // 最后的把可能进行变化的 HttpSession 之中的属性进行更新
        }

      }

      private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
      			ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

      		modelFactory.updateModel(webRequest, mavContainer); // 更新SessionAttribute values的值
      		if (mavContainer.isRequestHandled()) {
      			return null;
      		}
      		ModelMap model = mavContainer.getModel();
      		ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
      		if (!mavContainer.isViewReference()) { // to check whether this is a simple String of view name
      			mav.setView((View) mavContainer.getView());
      		}
      		if (model instanceof RedirectAttributes) {
      			// redirect: save add all attributes and direct to another
      			Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
      			HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
      			if (request != null) {
      				RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
      			}
      		}
      		return mav;
      	}
```

跟踪一下  **ModelFactory** 的使用过程：

在整个处理器方法被调用的过程中，ModelFactory只做了三件事情：

-  **getModelFactory** 生成一个ModelFactory
-  **ModelFactory.initModel** 在处理器方法调用之前进行模型数据的初始化
-  **ModelFactory.updateModel** 在处理器方法执行完成之后进行模型数据的更新 ，将 @SessionAttributes 注解声明的属性写入  **HttpSession** 之中

其实这样的话，已经可以解释为什么 @SessionAttributes 注解的属性在方法执行的时候，ModelMap已经将属性设置之后，Session 之中为什么还没有相应的属性了：
因为在实际的处理器方法执行完成之后的  **getModelAndView** 方法，才进行了模型的更新，将数据写入 Session 之中。

验证updateModel调用之前和之后的Session属性的区别：

> 1, 在之前的创建的项目中，找到  **ModelFactory.updateModel** ，打上断点：
>
> ![breakpoint](http://photos.zengxihao.xyz/f56c4853767e153099f51d07dbce5e2b.png)
>
> 2，运行项目，请求 http:localhost:8080/session/init
>
> 3，断点生效的时候，首先看一下session变量的值：
>
> ![breakpoint - variable values](http://photos.zengxihao.xyz/d0538527db49a36b9abb7484ae5f06d8.png)
>
> 4, 运行到this.sessionAttributesHandler.storeAttributes(request, defaultModel)之后，看下session属性：
>
> ![afterAttribuetStore](http://photos.zengxihao.xyz/126bf1da89ac8bd86fc657486bf1a5ec.png)
>
> 说明session属性的设置确实是在updateModel方法里面进行的，并且委托给 SessionAttributeHandler.storeAttributes 方法完成。
>

## 四，@SessionAttributes处理细节

### 1, ModelFactory 生成

getModelFactory的实现：

![getModelFactory](http://photos.zengxihao.xyz/c45467ccbbc4243a4c0af7d29ec72f5d.png)

主要做了四个事情：

- 调用getSessionAttributesHandler获取 SessionAttributesHandler
- 获取所有当前处理器类之中的带有 @ModelAttribute 注解的方法
- 先把全局的带有 @ControllerAdvice 注解的类中的 @ModelAttribute 注解方法加入执行方法列表	 **attrMethods** 中，然后把当前类的 @ModelAttribute 方法加入到列表中
- 生成一个ModelFactory 实例，并且返回

### 2, SessionAttributesHandler 生成

getSessionAttributesHandler的实现：

![getSessionAttributesHandler](http://photos.zengxihao.xyz/9681d5e6fa1fa325eea8d9c2a1163e87.png)

先从当前的缓存中尝试获取 SessionAttributesHandler 实例，没有的话自己new一个并且放入缓存。

用到了双判断null确保线程同步的技巧。

SessionAttributesHandler的构造函数：

![new SessionAttributesHandler](http://photos.zengxihao.xyz/406c4cfeee9efcc57e3eb675314ec7dd.png)

用@SessionAttributes注解的names和types属性来初始化成员属性  **attributeNames** 以及  **attributeTypes** ，最后还加入到  **knownAttributeNames** 已知的Session属性集合中。

### 3，modelFactory.initModel 实现

![initModel](http://photos.zengxihao.xyz/d89f566a65343abc2279e8dc44f73801.png)

- initModel首先把请求的Session之中的所有属性放到 ModelAndViewContainer 之中
- 然后对  **getModelFactory** 时候生成的 @ModelAttribute 注解的执行方法列表 进行调用，把结果也写入ModelAndViewContainer
- 最后把处理器方法带有  **@SessionAttribute** 的所有的参数找到，如果ModelAndViewContainer中还没有这个参数得话，从  **SessionAttributesHandler** 之中找到相应的值，找不到就抛出异常，找到了就保存进入ModelAndViewContainer之中。

### 4，SessionAttributesHandler.storeAttributes实现

上面说过，在ModelFactory的updateModel方法进行模型数据的更新，更新的操作是委托给 SessionAttributesHandler 完成的，而 SessionAttributesHandler 实现并不复杂，对于所有的模型数据，先判断符合是不是  **attrbuteNames** 或者  **attributeTypes** 之中指明的属性，两者是使用 @SessionAttributes 注解的属性进行初始化的，所以说也就是先判断是不是 @SessionAttributes 注解包含的属性，如果是的话，就调用  **SessionAttributeStore.storeAttribute** 方法保存进session之中，而  **SessionAttributeStore.storeAttribute** 的实现只是在属性的name之前加上前缀，但是默认的前缀是"" ，然后保存到request的session之中。

## 五，清除session中的 @SessionAttributes 注解加入的属性

通过 SessionStatus.setCompleted() 就可以清除@SessionAttributes加入session 之中的属性，实现比较简单：

![updateMpdel](http://photos.zengxihao.xyz/391c40b3c2df60a083666034a68ba1f9.png)

在updateModel 的实现中，如果判断sessionStatus isComplete ， 那么调用 SessionAttributesHandler.cleanupAttribute方法，

![cleanupAttributes](http://photos.zengxihao.xyz/bb02a1517d5494e4f8f79af8988c1f5e.png)

cleanupAttributes的实现 就是把之前记录的所有 @SessionAttributes 注解之中的属性，在request中一一删除，这样就达到了清除session属性的效果。

## 六，结尾

@SessionAttributes的属性注册以及对session进行更新都已经很清楚了，还有清除的实现也已经很清楚，其中还夹杂着对 @ModelAttribute 注解的方法的首先调用的解析。

实现的过程都不算很复杂，但是表达的逻辑很混乱，自己慢慢看其实就已经能看懂了的。
