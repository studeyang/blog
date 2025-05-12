---
permalink: 2024/0705.html
title: 
date: 2024-07-05 09:00:00
tags: 
cover: 
thumbnail: 
categories: technotes
toc: true
description: 文章模板，后面的文章可以参考这个模板的参数
---

通俗点来讲就是调用方是否需要等待结果，如果需要等待结果，就是同步；如果不需要等待结果，就是异步。

两种方式来实现异步：

第一，调用方创建一个子线程，在子线程中执行方法调用，这种调用我们称为异步调用；

```java
public void call() {
    new Thread(
        @Override
        public void run() {
            pai1M();
        }
    ).start();
    printf("hello world")
}
```

第二，方法实现的时候，创建一个新的线程执行主要逻辑，主线程直接 return，这种方法我们一般称为异步方法。

```java
public String call() {
    new Thread(
        @Override
        public void run() {
            // 逻辑代码
        }
    ).start();
    return "";
}
```

<!-- more -->

## 一、段落



## 二、段落



### 2.1 细分



### 2.2 细分



## 三、段落



![](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/202303052135542.gif)

## 封面



## 相关文章

- [学习分享（第3期）：你所理解的架构是什么？](https://mp.weixin.qq.com/s/ao9-DW3tXw25AW6D96m5LQ)

