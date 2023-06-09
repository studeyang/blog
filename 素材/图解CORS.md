---
permalink: 2022/1.html
title: 领域驱动设计与微服务
date: 2022-01-01 09:00:00
tags: 设计模式
cover: https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202305131823132.png
thumbnail: https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202305131823132.png
categories: template
toc: true
description: 文章模板，后面的文章可以参考这个模板的参数
---

CORS 的全称是 Cross-origin resource sharing，中文名称是跨域资源共享，是一种让受限资源能够被其他域名的页面访问的一种机制。

<!-- more -->

受限资源被其他域名的页面访问的机制，这句话中的 “其他域名的页面” 统称为源（Origin）。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/cors.png)



## 一、源（Origin）的定义

统一资源标示符（URI）用于标志某一互联网资源名，它的组成如下图所示。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230609103454720.png)

如果两个页面的协议（Protocol）、域名（domain）和端口（Port）都相同，则它们具有相同的来源。例如：

- `http://store.company.com/dir/page.html` 与 `http://store.company.com/dir2/other.html` ：同源
- `http://store.company.com/dir/page.html` 与 `https://store.company.com/secure.html	` ：不同源

不同源的访问请求叫做跨源请求（Cross Origin Requests），也就是通常说的跨域请求。通常情况下，跨域访问的请求会被浏览器拦截。

![image-20230609103515323](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230609103515323.png)

## 二、简单请求的跨域访问

若请求满足所有下述条件，则该请求可视为简单请求：

- 使用的是 `GET`、`HEAD`、`POST` 方法
- `Content-Type` 是 `text/plain`、`multipart/formdata`、`application/x-www-form-urlencoded`

### 2.1 如何进行跨域访问

如果想要实现跨域访问，可通过 HTTP 请求头 `Access-Control-Allow-Origin` 设置允许哪些域名跨域访问。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230609103538552.png)

这个 `Access-Control-Allow-Origin` 是服务器响应时添加到 HTTP 请求头里的，为了告诉浏览器这个响应数据允许这个源访问。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230609103611856.png)

### 2.2. 跨域发送身份凭证

当客户端与服务器端进行跨域请求时，如果需要在跨域请求中发送身份凭证（如 cookie 和 HTTP 认证信息），则需要在服务器端设置 `Access-Control-Allow-Credentials` 字段为 true，才能使客户端发送跨域请求时携带身份凭证。如果服务器端未响应 `Access-Control-Allow-Credentials` 或设置为 false，则浏览器会丢弃这个请求，从而导致无法进行跨域资源分享。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image-20230609102314695.png)



## 三、预检请求（Pre flight request）



Pre-flight request 可以翻译为“预检请求”，是跨域资源共享（CORS）机制中的一种预检机制。在进行跨域请求时，浏览器首先会发起一个 OPTIONS 请求，该请求称为预检请求，用于检查实际请求是否可以被服务器接受。预检请求中包含了实际请求将会用到的 HTTP 方法、头信息、请求地址等。

服务器在接收到预检请求后，会根据请求头中的 Origin 字段（表示请求发起的源地址）和请求内容，判断是否允许当前的跨域请求。如果允许，服务器会在响应头中添加 Access-Control-Allow-Origin 等 CORS 相关的信息，在实际请求中携带身份凭证（如 cookie）等信息，完成整个跨域请求过程。如果不允许，浏览器不会发起实际请求，并返回对应的错误响应码。

### 3.1 Access-Control-Allow-Methods


Access-Control-Allow-Methods 表示服务器允许的跨域请求的 HTTP 方法列表，如 GET、POST、PUT、DELETE 等。

### Access-Control-Allow-Headers

Access-Control-Allow-Headers 表示服务器允许的跨域请求的头信息列表，如 Authorization、Cache-Control、Content-Type 等。

### Access-Control-Max-Age

Access-Control-Max-Age 字段用于指定预检请求的缓存时间，单位为秒。一旦服务器端设置了 Access-Control-Max-Age 字段，浏览器在缓存期内会自动跳过预检请求，直接发起携带身份凭证的实际请求。这样可以降低服务器的压力，提升页面加载速度和用户体验。

![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202303052135542.gif)

## 封面



## Reference

- 统一资源标志符：https://zh.wikipedia.org/wiki/%E7%BB%9F%E4%B8%80%E8%B5%84%E6%BA%90%E6%A0%87%E5%BF%97%E7%AC%A6
- 跨域资源共享：https://zh.wikipedia.org/zh-hans/%E8%B7%A8%E4%BE%86%E6%BA%90%E8%B3%87%E6%BA%90%E5%85%B1%E4%BA%AB
- https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS

