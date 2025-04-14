---
permalink: 2024/0705.html
title: 
date: 2024-07-05 09:00:00
tags: 
cover: 
thumbnail: 
categories: technotes
toc: true
description: 文章模板，后面的文章可以参考这个模板 
---

自 Spring Boot 框架的流行，Tomcat 慢慢地被我们淡忘了。然而，Web 开发是离不开 Web 容器的，在定位程序问题时，就非常你的基础知识水平了。Tomcat 就是一项非常基础的技术框架。

<!-- more -->

## 一、什么是 Web 容器？

我们平时使用的微信小程序叫做 Applet，对这个单词简单翻译一下，可以理解为应用小程序。

那 Servlet 呢？有没有发现，这个单词跟 Applet 有点像。这样类比一下，Servlet 可以理解为运行在服务端的小程序，但是 Servlet 没有 main 方法，不能独立运行，因此必须把它部署到容器中，由容器来实例化并调用 Servlet。



## 二、Servlet





### 2.1 Servlet 规范



### 2.2 Servlet 容器



web.xml

```xml
<servlet>
    <servlet-name>spring-mvc</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>spring-mvc</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```



## 三、段落



![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202303052135542.gif)

## 封面



## 相关文章

- [学习分享（第3期）：你所理解的架构是什么？](https://mp.weixin.qq.com/s/ao9-DW3tXw25AW6D96m5LQ)

