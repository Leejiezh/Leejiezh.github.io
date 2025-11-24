---
title: springboot之监听器
top: false
cover: false
toc: true
mathjax: true
date: 2022-06-17 16:21:41
password:
summary:
tags: spring
categories: 后端
---

# springboot之监听器

ApplicationContext提供时间处理通过ApplicationEvent类和ApplicationListener接口，如果一个bean实现ApplicationListener接口在容器中，每次一个ApplicationEvent被发布到ApplcationContext中，这类bean就会收到这些通知。

实现Spring事件机制主要有4个类：

ApplicationEvent：事件，每个实现类表示一类事件，可携带数据。

ApplicationListener：事件监听器，用于接收事件处理时间。

ApplicationEventMulticaster：事件管理者，用于事件监听器的注册和事件的广播。

ApplicationEventPublisher：事件发布者，委托ApplicationEventMulticaster完成事件发布。



## 监听器使用

Web监听器的使用场景很多，比如监听servlet上下应用来初始化一些数据，监听HTTP Session 用来获取当前在线人数，监听客服端请求的ServletRequest 对象来获取用户的访问信息等等。

### 监听 Servlet 上下文对象

1. 首先我们要知道ServletContext的作用和详解

   ServletContext是一个全局的存储信息的空间，服务器开始就存在，服务器关闭就释放。request，一个用户可以有多个；session，一个用户一个；而servletContext是所有用户共用一个。

   运行在java虚拟机中的每一个web程序都有一个与之相关的Servlet上下文。一个ServletContext对象表示了一个web应用程序的上下文。

   Servlet上下文：Servlet上下文提供了对应用程序中所有Servlet所共有的各种资源和功能的访问。Servlet上下文API用于设置应用程序中所有Servlet共有的信息。Servlet可能需要共享他们之间的共有信息。运行于同一服务器的Servlet有时会共享资源，如JSP页面、文件和其他Servlet。



实践：[Spring Boot 中使用监听器](https://blog.csdn.net/taojin12/article/details/88338199)


