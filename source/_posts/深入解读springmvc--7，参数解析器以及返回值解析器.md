---
title: 深入解读springmvc--7，参数解析器以及返回值解析器
date: 2017-09-16 13:48:54
toc: true
comment: true
tags:
 - springmvc源码
---

## 一，引言

在大部分的情况下，我们使用@RequestMapping注解的处理器类或者处理器方法来进行请求的处理，处理器方法的参数经常会带有  **@ReqeustParam**,  **@RequestHeader** ,  **@CookieValue** 等等的注解，还有可能使用我们根本没有传入的参数，例如  **HttpServletRequest**,  **HttpServletResponse**,  **HttpSession**,  **ModelMap** 等，这些参数是怎么进行处理解析的，还有我们在处理器方法加上的  **@ResponseBody** ,  **@ResponseStatus** 等注解是怎么起到作用的。

处理的核心在使用 HandlerMapping 找到 HandlerMethod 之后，RequestMappingHandlerAdapter对于方法的调用的时候会使用一系列的  **HandlerMethodArgumentResolver** 和  **HandlerMethodReturnValueHandler** 的实现，对不同的注解和参数都会有相对应的解析器，例如比较常用的  **@ReqeustBody** 和  **@ResponseBody**  就会有一个  **RequestResponseBodyMethodProcessor** 对处理器方法传入的参数以及方法返回值进行相应的JSON解析 。

看一下调用的过程：

 **RequestMappingHandlerAdapter** 中的核心：

 <!--more-->

``` java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    ... // 一些局部变量的实例化

  	ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
  	if (this.argumentResolvers != null) {
      // 设置HandlerMethodArgumentHandlers
  		invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
  	}
  	if (this.returnValueHandlers != null) {
      // 设置HandlerMethodReturnValueHandlers
  		invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
  	}
  	invocableMethod.setDataBinderFactory(binderFactory);
  	invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

  	ModelAndViewContainer mavContainer = new ModelAndViewContainer();
  	mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request)); //get all request attributes

  	// invoke the @ModelAttribute methods and try to save the SessionAttribute into Model
  	// and guarantee @SessionAttribute values exist
  	modelFactory.initModel(webRequest, mavContainer, invocableMethod);

  	// redirect choices
  	mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

    ... // 异步请求的处理

  	invocableMethod.invokeAndHandle(webRequest, mavContainer); // 实际调用处理器方法

  	webRequest.requestCompleted(); // 最后请求完成的时候更新request

	}
```

很清楚的可以看到，所有的参数解析器以及返回值解析器都被设置到  **ServletInvocableHandlerMethod** 之中，然后全部委托给  **ServletInvocableHandlerMethod** 的  **invokeAndHandle** 方法，在实际调用处理器方法的时候再用这些解析器来进行解析。

## 二，解析器的调用流程

invokeAndHandle方法的实现：

``` java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs); // 执行方法获取返回值
		setResponseStatus(webRequest); // 设置Response Status

		...

		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
			this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest); // 处理返回值
		}
		catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
			}
			throw ex;
		}
	}

  public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
  			Object... providedArgs) throws Exception {

  		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);

  		Object returnValue = doInvoke(args);

  		return returnValue;
  	}
```

可以看到很清晰地分为三个步骤：
1. 用参数解析器解析参数
2. 反射执行方法
3. 用返回值解析器解析返回值

## 三，方法参数解析器

### 1，HandlerMethodArgumentResolver

HandlerMethodArgumentResolver定义了一个参数解析器需要实现的两个方法：

- 是否支持对该参数的解析
- 解析参数后的实际值，例如@ReqeustParam("name")String name 解析之后传给方法的实际参数值就是请求之中 name= xxx 的 xxx

![HandlerMethodArgumentResolver](http://ww1.sinaimg.cn/large/006pluSpgy1g0blnt435bj30rs0mpgo4.jpg)

springmvc 之中的 HandlerMethodArgumentResolver 其实可以分为四种：

- 对注解的解析
- 对类型的解析
- 自己定义的解析器
- 捕获所有的参数的解析器

看下springmvc提供的一系列的默认基于注解和类型参数解析器：

![default argument resolvers](http://ww1.sinaimg.cn/large/006pluSpgy1g0blo2mstrj30t60ko0vs.jpg)

catch-all 的参数解析器：

![catch-all argument resolvers](http://ww1.sinaimg.cn/large/006pluSpgy1g0bloal3csj30t103jt8t.jpg)

这些参数解析器大部分按照名字都可以猜出用途，一些具体的解析器可以自己看下实现。

### 2，基于注解的参数解析器 RequestParamMethodArgumentResolver

基于注解的参数解析器有很多，这里以  **RequestParamMethodArgumentResolver** 为例子进行分析：

处理参数的逻辑很简单，对于每一个处理器方法上面的参数，一一遍历所有的参数解析器找到支持的解析器，将参数解析。

ReqeustParamMethodArgumentResolver继承了  **AbstractNamedValueMethodArgumentResolver** ，  **AbstractNameValueMethodArgumentResolver** 是  **name - value - required flag** 类型的注解解析器的父类，定义了一个内部类  **NameValueInfo** 包含注解的信息，实现了  **HandlerMethodArgumentHandler** 接口的  **resolveArgument** 方法：

``` java
public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter); // to get the info of a certain parameter
		MethodParameter nestedParameter = parameter.nestedIfOptional(); // 支持Optional

		Object resolvedName = resolveStringValue(namedValueInfo.name); // 解析#{} 或者 ${} 的参数值

		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest); // resolveName 模板方法留给子类实现
		if (arg == null) {
			if (namedValueInfo.defaultValue != null) {
				arg = resolveStringValue(namedValueInfo.defaultValue); // 使用默认值
			}
			else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest); // 对于缺少参数的处理
			}
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveStringValue(namedValueInfo.defaultValue);
		}

		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			try {
				arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter); // 使用Binder 进行转换
			}
			catch (ConversionNotSupportedException ex) {
				throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
			catch (TypeMismatchException ex) {
				throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());

			}
		}

		handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest); // 模板方法

		return arg;
	}
```

可以看到对于使用占位符参数例如  **#{} 或者 ${}** 和 WebDataBinder进行参数转换的情况父类都已经做了处理，子类需要实现的只有：
1. 获取实际值的 resolveName 方法
2. 缺少参数时候的处理
3. 选择性地在参数解析完成之后进行一些处理

看下  **RequestParamMethodArgumentResolver** 中的  **supportsParameter** 实现：

![supportsParammeter](http://ww1.sinaimg.cn/large/006pluSpgy1g0blom39eqj30td0j50u2.jpg)

对于有 @RequestParam 注解的参数需要判断其中的name属性必须要有，如果没有的话使用的参数解析器是  **RequestParamMapMethodArgumentResolver** ，设置了默认类型的话基本类型的参数也是按照带有@ReqeustParam来进行处理的。

重写父类  **AbstractNamedValueMethodArgumentResolver** 的  **resolveName** 方法：

![resolveName](http://ww1.sinaimg.cn/large/006pluSpgy1g0blow7beoj30vr0joq4m.jpg)

处理的逻辑并不复杂：

先处理是文件上传的情况，然后调用request.getParameterValues(name)方法即可。

## 四，RequestResponseBodyMethodProcessor

**RequestResponseBodyMethodProcessor** 是对于  **@RequestBody** 的参数解析器以及  **@ResponseBody** 的返回值解析器，是SpringMVC之中比较常用的一个解析器。

### 1，继承关系

首先看一下  **RequestResponseBodyMethodProcessor** 的继承关系：

![ReqeustResponseBodyMethodProcessor的继承关系](http://ww1.sinaimg.cn/large/006pluSpgy1g0blp332n8j30s20ec0t9.jpg)

 **ReqeustResponseBodyMethodProcessor** 同时实现了  **HandlerMethodArgumentResolver** 以及  **HandlerMethodReturnValueHandler** 接口，其中类的作用如下：

| 类名                                       | 类的功能                                     |
| :--------------------------------------- | :--------------------------------------- |
| AbstractMessageConverterMethodArgumentResolver | 是所有的使用HttpMessageConverter从Http请求的body中解析参数的基类 |
| AbstractMessageConverterMethodProcessor  | 在 继承AbstractMessageConverterMethodArgumentResolver的基础上，加上了使用 HttpMessageConverter进行返回值的转换的功能 |
| RequestResponseBodyMethodProcessor       | 处理@RequestBody 注解的参数以及@ResponseBody的返回值，同时可以对@valid注解的参数进行验证 |

### 2，处理@RequestBody参数

supportsParameter的实现：

![supportsParameter](http://ww1.sinaimg.cn/large/006pluSpgy1g0blp93s7qj30m304fdfv.jpg)

简单判断一下参数是不是有@ReqeustBody注解

resolveArgument的实现：

![resolveArgument的实现](http://ww1.sinaimg.cn/large/006pluSpgy1g0blpexwo1j30v80hegnj.jpg)

逻辑很清晰：

1，用父类实现的  **readWithMessageConverters** 方法来转换参数为实际值
2，进行参数的验证
3，如果有 Optional 的话，将参数进行转换适配

父类实现的  **readWithMessageConverters** 包含了 在将参数转换为JSON对象之前自己定义的操作，以及使用 HttpMessageConverter 将参数进行转换，还有在参数转换之后的进行的操作。

### 3，处理@ResponseBody返回值

supportsReturnType的实现：

![supportsReturnType](http://ww1.sinaimg.cn/large/006pluSpgy1g0blq2ea9vj30ua06q74p.jpg)

首先判断一下类上是否带有@ResponseBody注解，没有的话再判断方法上面是否带有 @ResponseBody 注解。

handleReturnValue的实现：

![handleReturnValue](http://ww1.sinaimg.cn/large/006pluSpgy1g0blq7onpoj30te0a23ze.jpg)

处理流程：
1，设置请求已经处理完成的标志
2，调用父类实现的  **writeWithMessageConverters** 将返回值进行转换为 json 字符串。

父类实现的  **writeWithMessageConverters** 方法中，首先对 Http响应的  **MediaType** 进行处理，然后用支持转换的 HttpMessageConverter 进行转换。
并且对于实现了  **ResponseBodyAdvice** 的带有  **@ControllerAdvice** 的类，如果支持处理当前的请求，调用其实现的  **beforeBodyWrite** 方法，在返回值写入 响应之前进行自定义的处理。

### 4，HttpMessageConverter的实现

在 RequestMappingHandlerAdapter 之中，提供了一些已经实现的 HttpMessageConverter :

![default Http Message Converter](http://ww1.sinaimg.cn/large/006pluSpgy1g0blqddj2ej30ss089t9p.jpg)

最后的  **AllEncompassingFormHttpMessageConverter** 之中：

![判断是否存在相应的解析器](http://ww1.sinaimg.cn/large/006pluSpgy1g0blqiykkjj30vo0f0jt6.jpg)

判断当前的类路径下面是否存在一些包，存在的话就使用那些封装好的 JSON转换包提供参数转换的功能即可。

## 五，结尾

我感觉最近心态有点问题，心不在焉，看代码实在是太累了而且没有半点成就感，难得积累的一些兴趣都被消耗殆尽。

而且眼睛经常看电脑酸痛，可能是手机电脑玩太多了，身体好像有点累，需要锻炼才行了。

源码部分只剩下最后的视图渲染了，尽快完成这一系列的源码阅读吧。
