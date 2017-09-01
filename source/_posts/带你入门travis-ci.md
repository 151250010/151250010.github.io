---
title: 带你入门travis-ci
date: 2017-08-08 13:48:54
toc: true
comment: true
tags: travis-ci
---

## 一，travis-ci 简单介绍

travis-ci是专门为github项目提供服务的持续集成工具，支持大多数的主流语言，可以实现代码的自动编译，执行测试，测试结果报告和测试成功之后的自动部署。和jenkins类似，但是jenkins是开源的，支持自己定制化实现，并且不指定github，travis和jenkins比较还是差了许多的，但是项目的集成不是非常复杂并且使用github的话，travis不失为一个很好的选择。

<!--more-->

## 二，travis-ci 使用

### 1，注册使用TRAVIS

github 上面的public repository可以使用[https://travis-ci.org/][1]

private repository 使用[https://travis-ci.com/][2]

绑定github账号，然后选择需要使用的仓库：

![enter description here][3]

然后往仓库的根目录下面添加命名为.travis.yml的配置文件，然后进行push即可，多次commit的话会选择最新的一次进行构建。

### 2，自定义.TRAVIS.YML配置文件

首先需要了解一下travis的生命周期，travis一次构建由两个部分组成：install和script:

- install : 安装项目需要的各种依赖，包括系统需要的安装包，定制的服务等
- script: 执行构建脚本

一共可以细分为12个执行步骤，每个步骤都可以按照自己的需求进行定制化：

![enter description here][4]

每个步骤的具体解释可以看下文档：[https://docs.travis-ci.com/user/customizing-the-build/][5]

例如java项目的一个例子：

![enter description here][6]

sudo:required 表明需要的时候可以使用sudo，其他的选项为false,enabled
dist:trusty是改变默认的ubuntu 12.04版本升级到Ubuntu Linux Trusty 14.04

你也可以选择在mac系统中跑测试,

![enter description here][7]

### 3，执行SHELL脚本

每个步骤可以执行自己写的脚本来自定义构建：

``` yaml
install: 
	- ./install.sh
```

但是需要保证有运行脚本的权限，Linux系统可以使用chmod +x install.sh去除脚本的执行权限。

### 4，使用系统环境变量

travis提供了一些系统变量供使用，每次构建配置中我们可以使用这些变量：

变量具体说明： [https://docs.travis-ci.com/user/environment-variables#Default-Environment-Variables][8]

使用系统变量的例子:

``` yaml
after_success:
  - for name in $(find $TRAVIS_BUILD_DIR -name '*.html'); do ls -hl $name;  done
  - date
```

## 三，部署

travis-ci支持的部署环境很多，包括熟悉的AWS,github pages等，唯独少了阿里云，腾讯云等国内的云厂商。

AWS等的部署看下文档即可: [https://docs.travis-ci.com/user/deployment/][9]

对于国内的云服务器，我用expect配合ssh实现交互式的连接脚本，用scp传输需要的文件。

例如:

首先安装expect依赖:

``` yaml
before_install:
-sudo apt-get -qq update
-sudo apt-get install expect

```

然后执行shell脚本

![enter description here][10]

构建成功之后选择执行部署的脚本，然后在脚本中用完成部署的工作。

![enter description here][11]

但是讲账号密码明文写在脚本中是很不理智的，我们需要安装一个[the travis client][12]

按照README中的说明进行安装即可，首先执行`travis login --pro`或者`travis login` 进行登录，–pro时对应travis.com的私有仓库的

然后执行命令: `travis encrypt-file deploy.sh --add`即可，会输出一个deploy.sh.enc的加密之后的文件，而且会自动在.travis.yml文件中加上

![enter description here][13]

内容就是讲加密生成的deploy.sh.enc重新生成输出为deploy.sh，对于脚本我们需要去除运行权限，所以在生成deploy.sh之后，加上`chmod +x deploy.sh`即可：

![enter description here][14]

这样的话，就可以把原来的明文脚本去除，只剩下加密之后的脚本在我们的项目中，在安装依赖之前重新解密生成脚本。

## 四，结尾

其实travis还有很多东西可以选择去玩一下的，例如自定义notifications，还有深究怎么加快构建速度，这个文章权当个入门读物看下就好了。

（其实是写个博客居然花了我两个小时，，实在不想写下去了，到这里也确实算是入门了，我有深入的使用感悟之后下次还是会写博客的。）

  [1]: https://travis-ci.org/
  [2]: https://travis-ci.com/
  [3]: /assets/images/带你入门travis-ci/1504200923703.jpg
  [4]: /assets/images/带你入门travis-ci/1504200967720.jpg
  [5]: https://docs.travis-ci.com/user/customizing-the-build/
  [6]: /assets/images/带你入门travis-ci/1504200997169.jpg
  [7]: /assets/images/带你入门travis-ci/1504201024217.jpg
  [8]: https://docs.travis-ci.com/user/environment-variables#Default-Environment-Variables
  [9]: https://docs.travis-ci.com/user/deployment/
  [10]: /assets/images/带你入门travis-ci/1504201237775.jpg
  [11]: /assets/images/带你入门travis-ci/1504201251349.jpg
  [12]: https://github.com/travis-ci/travis.rb#readme
  [13]: /assets/images/带你入门travis-ci/1504201329895.jpg
  [14]: /assets/images/带你入门travis-ci/1504201369749.jpg