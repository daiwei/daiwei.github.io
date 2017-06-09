---
layout: post
title: clang-layered-system开源
---

**TL;DR**   这份代码主要是运用C语言进行模块化开发的一个试验框架


# 概述

现将之前所做的[clang-layered-system](https://github.com/daiwei/clang-layered-system)开源，这里只是个框架demo。


# 模块化

将一个独立模块封装到一个动态库中，需要给外部调用的接口使用EXPORT导出，外部使用时简单的使用IMPORT即可。

模块间使用DBus作为消息传递的通道。


# 分层

- app
- framework
- service
- hardware


# 模块间通信

- 基于DBus


# 通信分类

- method           // module -> module
- command          // loader -> module
- notify           // broadcast
- push             // backend -> system/user/module
