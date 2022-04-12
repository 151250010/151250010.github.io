---
title: 动态字段的注入
date: 2021-05-03 13:48:54
toc: true
comment: true
tags:
- Java
- 字节码
---

## 一，概述

本文主要分析基于运行时给java对象添加动态字段的多种实现原理及实现手段。

### 1.1 什么是动态字段

![动态字段示意图.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1610854172421-fa016c3b-5973-4e86-9c6e-452c40c72b99.png#crop=0&crop=0&crop=1&crop=1&height=412&id=WUByf&margin=%5Bobject%20Object%5D&name=%E5%8A%A8%E6%80%81%E5%AD%97%E6%AE%B5%E7%A4%BA%E6%84%8F%E5%9B%BE.png&originHeight=412&originWidth=972&originalType=binary&ratio=1&rotation=0&showTitle=false&size=47342&status=done&style=none&title=&width=972)

如图所示：

核心要实现的功能是： 基于属性上的注解，在运行时动态生成一个新的字段。Person.age 在运行时基于age的值，动态新增了ageDesc属性和其属性值。这样就支持了在运行时动态注入新字段。

这种情况其实并不多见，这样会破坏了代码的可读性，提供给外部的接口会新增部分的字段，而这些字段不是直接用类属性定义，而是在某个类属性的枚举上声明的。

### 1.2 我们使用的业务场景

我们在生产环境对于这个工具的使用情况是：

针对形如：
```json
BizType(1000, "我是1000业务")
BizType(2000, "我是2000业务")
BizType(3000, "我是3000业务")
```

大致如上的枚举数据，而且其枚举解释可能会动态调整。针对这种情况接入了基础域的配置中心，动态生成了"bizDesc"字段，然后对外暴露的字段的释义统一在配置中心进行维护。

我们在接入配置中心过程中遇到了一个问题。配置中心的这个动态字段功能对于spring mvc框架无法支持，只在一个自用的MVC框架中做了支持，所以研究了一下这个动态字段的实现，并且补充了其在spring mvc框架的动态字段功能实现。

综上说明了动态字段是什么，及我们使用的业务场景，下面探讨一下多种实现方案的原理和核心代码。

## 二，实现方法

实现方法都是以开始图中的Person对象为例，阐述在Person对象中新增ageDesc字段的多种实现方式。

```java
@Data
@Builder
public class Person {

    @ConfigValue(group = "xxx.group", produces = "nameDesc")
    private String name;

    @ConfigValue(group = "xxx.group", produces = "ageDesc")
    private Integer age;

    public Person() {
    }

    public Person(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
}
```

### 2.1 字节码增强

遇到新增属性的场景，一般都会想通过字节码增强的方法来从根本上新增类属性。

原理： 通过字节码增强的手段，手动给带有相应注解属性的类的字节码文件做增强，增加需要的字段和对应的字段getter，setter方法。字节码增强就是修改类编译出来的字节码文件(.class)，从而做到不知不觉中修改类的定义。

而在什么时候修改类的字节码，怎么做到修改类的字节码都有多种选择。

可以在代码编译的时候通过java编译器开放的扩展点进行源代码的修改。也可以在运行时配合agent进行动态字节码数据修改。

#### 2.1.1 JSR-269规范及动态字段实现

java代码通过javac编译器编译成jvm字节码(.class文件)的过程：

> 图引自网络

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1612098517343-13c29dba-bd65-467c-9612-ceabf6cc874e.png#crop=0&crop=0&crop=1&crop=1&height=235&id=YXAV5&margin=%5Bobject%20Object%5D&name=image.png&originHeight=235&originWidth=836&originalType=binary&ratio=1&rotation=0&showTitle=false&size=268916&status=done&style=none&title=&width=836)


jvm字节码被虚拟机执行过程：

jvm字节码会被翻译成机器可识别的机器语言，一种是解释执行，即逐条将字节码翻译成机器码执行；一种是即时编译，即将一个方法中包含的所有字节码编译成机器码后再执行。

> 图引自网络

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1612098681774-30a9024c-6cfb-43d7-8e53-9f6c87083c36.png#crop=0&crop=0&crop=1&crop=1&height=404&id=xB0uH&margin=%5Bobject%20Object%5D&name=image.png&originHeight=404&originWidth=720&originalType=binary&ratio=1&rotation=0&showTitle=false&size=85339&status=done&style=none&title=&width=720)

上面两张图解释一下java源代码(.java文件)通过编译生成字节码文件(.class文件)，字节码文件再通过解释执行器/即时编译器的过程。具体的编译原理和jvm字节码翻译实现，我也不了解，后续可以找点demo学习一下。

JSR269规范是在代码编译成字节码文件过程，产生了注解抽象语法树之后留的一个扩展点，是一套基于注解的规范，允许开发者在最终生成jvm字节码文件之前，做一些修改。这样的话我们用注解就能玩出花来了。基于方法的注解和JSR269规范，可以做到无侵入的偷偷在方法执行前植入一段代码，修改方法的定义，方法返回值都可以轻松做到。lombok， mapstruct都是通过这个规范来做到动态修改代码的，

JSR269规范：

> Java6开始纳入了JSR-269规范：Pluggable Annotation Processing API(插件式注解处理器)。JSR-269提供一套标准API来处理Annotations，具体来说，我们只需要继承AbstractProcessor类，重写process方法实现自己的注解处理逻辑，并且在META-INF/services目录下创建javax.annotation.processing.Processor文件注册自己实现的Annotation Processor，在javac编译过程中编译器便会调用我们实现的Annotation Processor，从而使得我们有机会对java编译过程中生产的抽象语法树进行修改 。


以要新增字段ageDesc , nameDesc为例，只需要在源代码编译过程中新增名称为getAgeDesc的方法即可。具体的
JSR269规范使用，和语法树的相关api可以自行查阅，只要继承抽象类javax.annotation.processing.AbstractProcessor就行，直接贴一下基于注解调用基础域接口，新增getAgeDesc的代码：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1612100398409-4e1b58c4-df45-4d1d-a097-3e4eb7c88e75.png#crop=0&crop=0&crop=1&crop=1&height=1308&id=QGa9y&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1308&originWidth=1400&originalType=binary&ratio=1&rotation=0&showTitle=false&size=1145123&status=done&style=none&title=&width=1400)


新增方法需要告诉编译器遇到注解要用这个注解解析器：通过META-INF/services/javax.annotation.processing.Processor 这样的一个文本文件就可以告知编译器了。
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1612100645612-59455236-40a1-4241-9f49-299cbda96ba4.png#crop=0&crop=0&crop=1&crop=1&height=308&id=wTrXX&margin=%5Bobject%20Object%5D&name=image.png&originHeight=308&originWidth=747&originalType=binary&ratio=1&rotation=0&showTitle=false&size=100554&status=done&style=none&title=&width=747)

这样就可以了，打一个包，然后让第三方依赖使用，就会基于注解自动在编译时候新增getXXX()方法了, 下面是验证结果。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1612101113721-abd43ae7-98d3-49ae-b5a4-b87327450941.png#crop=0&crop=0&crop=1&crop=1&height=170&id=MFXAe&margin=%5Bobject%20Object%5D&name=image.png&originHeight=170&originWidth=612&originalType=binary&ratio=1&rotation=0&showTitle=false&size=83154&status=done&style=none&title=&width=612)

使用到了@DictValue注解的应用，直接pom引入processor的依赖，mvn clean compile，然后在编译出来的目录
${baseDir}/target/classes 下找一下Person类编译出来的字节码文件：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1612101306586-88ea0bde-9bf9-43d0-8dfb-817d542e3942.png#crop=0&crop=0&crop=1&crop=1&height=1160&id=b2lwx&margin=%5Bobject%20Object%5D&name=image.png&originHeight=1160&originWidth=1508&originalType=binary&ratio=1&rotation=0&showTitle=false&size=848625&status=done&style=none&title=&width=1508)

下面说一下开发遇到的问题和解法，需要自取：

- processor包需要先编译通过，然后才能指定processor来编译对应的代码，maven打包需要在打包resources文件的时候加上执行参数：
```xml
<executions>
                    <execution>
                        <id>process-META</id>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>copy-resources</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>target/classes</outputDirectory>
                            <resources>
                                <resource>
                                    <directory>${basedir}/src/main/resources/</directory>
                                    <includes>
                                        <include>**/*</include>
                                    </includes>
                                </resource>
                            </resources>
                        </configuration>
                    </execution>
                </executions>
```

- 构造方法的时候，如果代码有静态方法调用，引入静态方法的时候比较复杂，需要在语法树里一层一层获取最后执行的方法节点，以com.xxxx.ascm.kv.configer.client.DictUtil.getDictValue为例，需要用这个代码才能准确调用到方法：不要傻傻的用字符串"com.xxxx.ascm.kv.configer.client.DictUtil.getDictValue"去调用方法
```java
expression = com.sun.tools.javac.tree.TreeMaker#Select("com", "xxxx")
expression = com.sun.tools.javac.tree.TreeMaker#Select(expression, "ascm")
expression = com.sun.tools.javac.tree.TreeMaker#Select(expression, "kv")
expression = com.sun.tools.javac.tree.TreeMaker#Select(expression, "configer")
expression = com.sun.tools.javac.tree.TreeMaker#Select(expression, "client")
....
expression = com.sun.tools.javac.tree.TreeMaker#Select(expression, "getDictValue")

```

ps: 全部jsr规范列表： [https://jcp.org/en/jsr/all](https://jcp.org/en/jsr/all) , 有空可以全部看看， 翻译一下。

#### 2.1.2 通过agent运行时动态修改字节码

可以参考美团文档： [https://tech.meituan.com/2019/11/07/java-dynamic-debugging-technology.html](https://tech.meituan.com/2019/11/07/java-dynamic-debugging-technology.html)

这个没有做实现。可以简单说一下这个实现的原理，和涉及到的api。建议大家可以动手玩玩。

上面的实现是在编译时生成jvm字节码文件之前，做的代码编译的语法数的修改， 通过agent的形式，是可以在生成了字节码之后，运行时动态改掉字节码的内容。熟悉的Arthas(阿尔萨斯) 就是用这种方式来hack我们的代码的，idea的debug也是通过这种原理，允许我们在evaluate expression的时候随便读写内存数据。

简单阐述一下原理：

通过在jvm启动时通过形如 javaagent 命令注册 或者运行时通过 VirtualMachine attach, loadagent 这些接口进行
agent的引入。通过java agent技术进行类的字节码修改最主要使用的就是Java Instrumentation API。

> instrument是JVM提供的一个可以修改已加载类的类库，专门为Java语言编写的插桩服务提供支持。它需要依赖JVMTI的Attach API机制实现。在JDK 1.6以前，instrument只能在JVM刚启动开始加载类时生效，而在JDK 1.6之后，instrument支持了在运行时对类定义的修改。要使用instrument的类修改功能，我们需要实现它提供的ClassFileTransformer接口，定义一个类文件转换器。接口中的transform()方法会在类文件被加载时调用，而在transform方法里，我们可以利用ASM或Javassist对传入的字节码进行改写或替换，生成新的字节码数组后返回。



![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1616587233335-b83b40fd-f241-4abe-8195-97985400bcd0.png#crop=0&crop=0&crop=1&crop=1&height=763&id=lVXX2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=763&originWidth=1200&originalType=binary&ratio=1&rotation=0&showTitle=false&size=115423&status=done&style=none&title=&width=1200)



### 2.2 MVC层序列化数据注入

还有一种实现手段，可以在MVC框架的message converter层进行字段注入，以通用的Spring MVC来说明，可以在进行对象转换的view层，使用jackson进行数据序列化的时候动态注入新的字段。

同理，对于使用Gson进行序列化的MVC框架，Gson也是可以完美支持字段注入的。

核心原理：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1616587340078-96746104-c064-4983-8c80-98b6d767c449.png#crop=0&crop=0&crop=1&crop=1&height=513&id=tK8gR&margin=%5Bobject%20Object%5D&name=image.png&originHeight=513&originWidth=800&originalType=binary&ratio=1&rotation=0&showTitle=false&size=104890&status=done&style=none&title=&width=800)

在spring mvc做模型渲染成JSON数据的时候，可以通过渲染JSON工具开的扩展点，做到动态往渲染出来的JSON数据中新增属性。

配置中心使用的是该种实现，其实现有局限性：动态新增的字段只在web层生效。。。

#### 2.2.1 Jackson序列化注入数据

实现简述，jackson框架支持扩展的，Jackson也定义了一堆的自己的注解，如JsonField , JsonAppend等等，这些注解在属性上使用，然后jackson序列化数据时候会识别注解，动态修改属性值或者名称。

可以找到Jackson支持注解的相关类实现，进行新注解支持和扩展即可。在序列化这个阶段动态注入我们的新属性。

核心的代码实现及API：

**com.fasterxml.jackson.databind.introspect.NopAnnotationIntrospector.findAndAddVirtualProperties**
这个是Jackson的扩展实现， 通过继承覆盖该方法可以做到在序列化数据过程中，动态注入虚拟节点。从而实现动态web字段的功能。

然后配置一下ObjectMapper即可：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1616587800213-3824e0d6-433a-4bac-8907-8dccd85619cb.png#crop=0&crop=0&crop=1&crop=1&height=445&id=M7wx8&margin=%5Bobject%20Object%5D&name=image.png&originHeight=445&originWidth=1726&originalType=binary&ratio=1&rotation=0&showTitle=false&size=403918&status=done&style=none&title=&width=1726)

#### 2.2.3 Gson序列化注入数据

Gson也留了扩展实现，在GSON序列化数据过程中可以自定义某类型数据的渲染，可以直接继承com.google.gson.FieldAdapterFactory类来实现。

然后将自定义的FieldAdapterFactory注册到Gson解析器中即可：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2882184/1616588017095-b51b48ac-a6ad-4a4f-a5a1-52fab855fc0e.png#crop=0&crop=0&crop=1&crop=1&height=237&id=CSnuk&margin=%5Bobject%20Object%5D&name=image.png&originHeight=237&originWidth=987&originalType=binary&ratio=1&rotation=0&showTitle=false&size=142036&status=done&style=none&title=&width=987)

## 三，总结

无。

把做业务过程中难得的技术问题记录一下。分享一下运行时注入新属性的多种实现原理及实现手段。

各取所需吧～ 
