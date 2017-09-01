---
title: 深入解读springmvc--1，摸清DispatcherServlet
date: 2017-08-13 13:48:54
toc: true
comment: true
tags:
 - springmvc源码 
 - custom spring mvc
---

## 一，前言

很惭愧最近没更新博客，闲的慌废寝忘食地看了几本小说，还好一周时间就看吐了，上周空闲之余粗略看了下Spring MVC的源码，先把上周的一些思路整理一下，走一下springmvc的流程。

打算写一个springmvc系列的博客，从源码解读，到自己实现一个处理请求，生成视图和模型并且可以进行视图的解析的A货MVC框架。DispatcherServlet是整个SpringMvc框架的核心，是整个web应用程序的入口点，捕获了所有的请求，进行分析找到相应的控制器和方法来处理这个请求，然后生成模型和视图，最后由视图来进行渲染自身。

>整篇博客分为三个部分：

- 了解Java Servlet
- DispatcherServlet的继承关系
- DispatcherServlet初始化做的每一步的工作

>下一篇博客会继续分析DispatcherServlet:

- mvc处理Http请求的具体流程
- 自己实现一个A货DispatcherServlet

<!--more-->

## 二，了解Java Servlet

什么是Servlet呢，servlet是一个java类，是服务器端运行的一个小程序，并且看可以通过**请求-响应**编程模型来访问的服务器内存中的程序。

java实现servlet的三种方式：

### 1，java定义好了servlet的接口，我们直接实现其即可：

![enter description here][1]

五个方法：

- init: servlet被Servlet容器实例化的时候就调用，如果init方法抛出ServletException或者是没有在规定时间内return那么容器就会直接忽视这个Servlet
- getServletConfig: 获得Servlet初始化的配置
- service: 被Servlet容器调用进行请求的处理
- getServletInfo: 获取Servlet的信息
- destroy: 当Servlet服务完成之后由容器调用，进行资源的释放等操作

实现了Servlet接口，就可以放在Servlet容器中，在web.xml配置中配置Servlet和其对应的响应路径即可。给出一个简单的例子：

项目结构：

![enter description here][2]

web.xml配置：

``` xml
<servlet>
     <servlet-name>xihao</servlet-name>
     <servlet-class>com.test.xihao.XihaoWeb</servlet-class>
 </servlet>
 
 <servlet-mapping>
     <servlet-name>xihao</servlet-name>
     <url-pattern>/xihao</url-pattern>
 </servlet-mapping>
```
XihaoWeb就是一个简单实现了Servlet接口的类，然后把整个项目部署到tomcat中即可。

### 2，继承GenericServlet

GenericServlet是一个通用的，不依赖具体协议的Servlet，实现了Servlet,ServletConfig接口，还实现了log方法，只需要继承GenericServlet，实现一个抽象方法service即可。

然后在web.xml中声明其映射。

### 3，继承HttpServlet

具体的Http协议的实现，继承了GenericServlet，最通用的一种实现servlet的方式就是直接继承HttpServlet然后重写其中的:

- doGet
- doPost
- doPut
- doDelete
- init
- getServletInfo
等方法即可。

## 三，DispatcherServlet的继承关系

springmvc源码错综复杂，就是一个庞大的蜘蛛网，我们的思路一定要清晰，先从DispatcherServlet的继承关系下手，看下其继承关系:

![enter description here][3]

> HttpServletBean

HttpServletBean 实现了EnvironmentCapable,EnvironmentAware接口，可以自定义web运行的环境，继承HttpServlet，主要需要注意的是初始化的时候完成了web容器的依赖注入：

``` java
public final void init() throws ServletException {
		if (logger.isDebugEnabled()) {
			logger.debug("Initializing servlet '" + getServletName() + "'");
		}
		// Set bean properties from init parameters.
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
				//从ServletContext中获取Bean属性参数，然后进行web容器的依赖注入
				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
				initBeanWrapper(bw);
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
				if (logger.isErrorEnabled()) {
					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
				}
				throw ex;
			}
		}
		//子类自己实现ServletBean具体的初始化
		initServletBean();
		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
	}
```

> FrameworkServlet

SpringMvc框架的基层Servlet，提供了在bean层面上与Spring应用上下文的集成，主要实现了三个功能：

 - 基于每个Servlet的命名空间中的beans，为每个Servlet实例化一个WebApplicationContext，并对其进行管理
 - 在处理请求上发布事件，不管请求是否成功
 - 完成了Servlet上下文的建立之后，子类实现onRefresh方法，完成IOC容器的初始化

子类必须覆盖其doService方法实现具体的请求处理逻辑。

> DispatcherServlet

作为一个前端控制器，所有的web请求都需要通过它进行处理，进行转发，匹配，数据处理后，并由页面进行处理。是SpringMvc的核心。

主要实现的功能可以分为两个部分：

- 初始化部分，主要在initWebApplicationContext中，对web模块需要的bean进行了初始化
- 对Http请求进行响应，主要在doDispatch中实现

## 四，DispatcherServlet的执行链

DispatcherServle怎么样处理Http请求的呢，还有初始化的时候具体做了些什么呢？

看下两条主线：

DispatcherServlet自身IOC容器的初始化：

![enter description here][4]

处理Http请求：

![enter description here][5]

### 1，DISPATCHERSERVLET的初始化

DispatcherServlet自己持有一个IOC容器，整个web应用有一个根上下文，在ContextLoaderListener初始化的时候被建立了，并且被设置到WebApplicationContextUtils中去。

然后可以理解为每个Servlet持有一个子上下文，DispatcherServlet的持有的WebApplicationContext以web应用的根上下文作为自己的parent(这个过程在FrameworkServlet中实现)。需要知道的是，在向IOC容器getBean的时候，首先会到双亲上下文中寻找，也就是说，整个web项目的根上下文定义的bean是可以被所有的子上下文共享的。

具体看下FrameworkServlet中的initServletBean方法，以及一些注释：

``` java
protected WebApplicationContext initWebApplicationContext() {
		//获取根上下文,使用这个作为当前mvc上下文的双亲上下文
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext()); 
		WebApplicationContext wac = null;
		if (this.webApplicationContext != null) {
			 //构造函数时候已经注入了web应用上下文
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						//没有双亲上下文的话，设置上下文,可能为null
						cwac.setParent(rootContext);
					}
					//对子上下文进行基本配置和最终调用IOC容器的refresh方法
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			// 没有的话 就去ServletContext中找,找到的话 默认按照根上下文已经被设置并且用户已经执行了初始化来处理
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// 最后还是没有找到的话，创建一个
			wac = createWebApplicationContext(rootContext);
		}
		if (!this.refreshEventReceived) {
			// 还没有进行刷新事件的话
			// 如果创建的上下文不支持刷新或者是构造器注入的上下文已经被调用过refresh方法，在这里手动刷新
			onRefresh(wac);
		}
		if (this.publishContext) {
			//把生成的子上下文设置到SevletContext中去
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
			}
		}
		return wac;
	}
```

**FrameworkServlet**中的onRefresh方法是空的，是一个模板留给子类实现。然后**DispatcherServle**t具体实现了onRefresh方法，调用自身的initStrategies方法，看下其具体实现：

![enter description here][6]

initStrategies(ApplicationContext context)就是mvc框架的各种实现元素的初始化，包括国际化的LocaleResolver，配置请求和controller映射的HandlerMappings等。

上面的流程还有最后的两步没有打通，哪两步呢：

- configureAndRefreshWebApplicationContext方法和createWebApplicationContext的具体实现
- 最后的以initHandlerMapping方法为例，最后的mvc组件的具体实例化又是怎么样的呢？

> 两个方法解析：

首先看下createWebApplicationContext:

![enter description here][7]

设置一些基本属性之后还是调用的configureAndRefreshWebApplicationContext方法，看下configureAndRefreshWebApplicationContext:

![enter description here][8]

其实就是设置一些基本的参数之后，然后进行容器的所有bean的实例化。

> initHandlerMapping具体解析：

![enter description here][9]

看下默认的配置，DispatcherServlet.properties：

![enter description here][10]

可以看到默认的HandlerMappings的具体实现是BeanNameUrlHandlerMapping和RequestMappingHandlerMapping。

- BeanNameUrlHandlerMapping 基于bean名字来匹配映射，举个例子就是有个/userName请求，会查找controller的name属性为”/userName”的controller，支持模糊匹配，”/user*”也可以匹配到”/userName”的controller。
- RequestMappingHandlerMapping 应该是最常用的一个解析器了吧，请求直接映射到@RequestMapping注解的类和方法上面。

## 五，结尾

写到这里已经是深夜的00：30了，这种源码阅读的操作简直就是在啃书一样，太过于晦涩了，错综复杂的调用关系以及层层继承下来，需要时刻保持头脑的高度清晰和集中。

但是现在，整个DispatcherServlet初始化的步骤已经很明朗，流程贯通的话，还是有一点点的成就感的，清楚了各个步骤之后，我们就可以自己定制一个DispatcherServlet了，也可以按照自己的需求进行框架的深度定制，这些操作留到下一篇博客，给自己找一些成就感哈哈。

  [1]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504201913380.jpg
  [2]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504201950709.jpg
  [3]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202068809.jpg
  [4]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202306851.jpg
  [5]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202316825.jpg
  [6]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202423436.jpg
  [7]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202520975.jpg
  [8]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202531347.jpg
  [9]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202562066.jpg
  [10]: /assets/images/深入解读springmvc--1，摸清DispatcherServlet/1504202572563.jpg