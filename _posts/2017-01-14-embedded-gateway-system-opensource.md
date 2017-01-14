---
layout: post
title: embedded-gateway-system开源
---

**TL;DR** 这份代码主要是运用C语言进行模块化开发的一个试验框架


# 概述
现将之前所做的[embedded-gateway-system](https://github.com/daiwei/embedded-gateway-system)开源，这里只是个框架demo。


# 说明
## 模块化
将一个独立模块封装到一个动态库中，需要给外部调用的接口使用EXPORT导出，外部使用时简单的使用IMPORT即可。
模块间使用DBus作为消息传递的通道。


## 分层
1. app
2. framework
3. service
4. hardware


## 模块间通信
1. 基于DBus


## 通信方式
1. method           // module -> module
2. command          // loader -> module
3. notify           // broadcast
4. push             // backend -> system/user/module


