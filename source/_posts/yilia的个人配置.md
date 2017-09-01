---
title: yilia的个人配置
date: 2017-08-05 13:48:54
toc: true
comment: true
tags: 建站教程
---

## 一，theme配置文件的修改

yilia的主要配置在themes/yilia/_config.yml中，我们对其进行需要的修改即可。

![enter description here][1]

可以参考主题作者的配置：[https://github.com/litten/hexo-theme-yilia][2]

还有的是主题作者的博客备份:[https://github.com/litten/BlogBackup][3]

<!--more-->

### 1，配置自己的头像

修改themes/yilia/_cofig.yml 中的头像url路径，

![enter description here][4]

/assets/image/icon.jpg 对应的是blog/source/assets/image/icon.jpg

![enter description here][5]

### 2，配置ABOUTME

找到/themes/yilia/_config.yml中的最后aboutme,按照自己需要填写自己的简介即可。：

![enter description here][6]

### 3，配置其他的东西

其实在项目的readme中都有很详细的说明，可以自己按需配置，不懂配置的可以留言

## 二，集成第三方服务

### 1，集成畅言评论

yilia主题提供了多种评论服务的集成：

- 多说
- 网易云跟帖
- 畅言
- Disqus

但是多说已经关闭服务了，网易云跟帖好像有限期的，我选择了集成畅言评论：

畅言注册一个账号之后需要配置站点的信息才可以使用，而且需要有网站的备案：

![enter description here][7]

- 站点名称是给自己的网站起个名字
- 站点网址是和ICP备案号相匹配的（没有备案号的下面说下怎么获取一个）
- 域名白名单是允许有评论功能的网站的主域名，可以配置多个主域名，只要上面的备案通过了第一个写备案网站，后面写自己的github page就可以了

#### 获取网站的备案号

给出网站查询百度备案号的链接，自己随便找个网上的域名来搜一下即可

[http://www.beianbeian.com/gaoji?field=siteDomain&value=baidu.com][8]

获取到备案号 和站点网址一起写到设置中：

需要注意的是站点网址要填具体的http例如：

![enter description here][9]

最后点确认，需要等待几个小时验证通过

#### 获取生成的appid和conf

如果顺利的话，通过验证了会生成一段代码大概如图：

![enter description here][10]

找到里面的appid,conf两个字符串，copy填到/themes/yilia/_config.yml中:

![enter description here][11]

不要使用我的id，因为我的白名单里面没有你们的github.io，所以你们copy我的appid是不会成功的。

修改配置文件之后，执行

- hexo clean
- hexo g -d

等待一会儿，打开自己的github page就可以看到评论功能了！

### 2，集成百度分析

打开百度分析网页选择注册站长账号: [https://tongji.baidu.com/web/register][12]

按信息填写即可，成功之后会生成一段的密钥，直接copy到/themes/yilia/_config.yml中的baidu_analytics中即可:

![enter description here][13]

## 三，结尾

感觉到这里，一个功能完整的博客雏形已经出来了，可以试下自己写博客，然后部署上传，看到自己写的博客是不是满满的成就感呢，哈哈哈。

写博客我是用的[小书匠][14]，还是很不错的，qq的截图直接可以粘贴进文档之中，记住一些快捷键的话使用起来相当便捷。（吐槽的是页面功能不突出，找自己保存的文档找了半天没找到。。）


  [1]: /assets/images/yilia的个人配置/1504195372695.jpg
  [2]: https://github.com/litten/hexo-theme-yilia
  [3]: https://github.com/litten/BlogBackup
  [4]: /assets/images/yilia的个人配置/1504195584297.jpg
  [5]: /assets/images/yilia的个人配置/1504195602109.jpg
  [6]: /assets/images/yilia的个人配置/1504198005053.jpg
  [7]: /assets/images/yilia的个人配置/1504198219512.jpg
  [8]: http://www.beianbeian.com/gaoji?field=siteDomain&value=baidu.com
  [9]: /assets/images/yilia的个人配置/1504198294481.jpg
  [10]: /assets/images/yilia的个人配置/1504200392647.jpg
  [11]: /assets/images/yilia的个人配置/1504200403632.jpg
  [12]: https://tongji.baidu.com/web/register
  [13]: /assets/images/yilia的个人配置/1504200480400.jpg
  [14]: http://markdown.xiaoshujiang.com/