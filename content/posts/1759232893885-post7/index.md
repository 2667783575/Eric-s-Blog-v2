---
date: '2025-08-29T16:43:20+08:00'
author: "Eric"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "Interface Segregation Principle"
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
title: "OODP3 接口隔离原则(ISP)"
categories: "技术相关"
tags: ["OODP","C#"]
---

# 不应强迫客户端依赖它不使用的方法

该原则过于简单，索性不放代码

## 例子

一个现代打印机可以打印(Print),传真(Fax),扫描(Scan)，所以我们写了一个接口IMachine，并让它包括这三个功能的函数

然而，当我们需要创建一个老式打印机时，该打印机只有打印功能，如果实现IMachine接口，Fax和Scan两个函数要么抛出异常，要么打印未实现，这都是非常不好的情况(即我们强迫老式打印机依赖了它不使用的方法)。

因此我们要把IMachine接口拆分开，如IPrinter,IFaxer,IScanner，以使得客户端需要使用什么方法就实现什么接口。

## 和SRP的区别

| 特性           | SRP (单一职责原则)                                                                                                    | ISP (接口隔离原则)                                              |
| :------------- | :-------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------------- |
| **关注点**     | **类的内部凝聚力** - 一个类应该只做一件事                                                                             | **接口的外部耦合** - 一个接口不应该强迫客户端依赖它不需要的方法 |
| **主要对象**   | **类**（Class）                                                                                                       | **接口**（Interface）                                           |
| **核心思想**   | **只有一个变化原因**                                                                                                  | **不强迫客户接受不需要的方法**                                  |
| **解决的问题** | 类过于庞大、僵化、难以修改                                                                                            | 接口臃肿，导致客户端实现不必要的空方法或收到“接口污染”          |
| **关系**       | **相辅相成**。通常，遵循SRP拆分出的类，自然会拥有更单一、更聚焦的接口。而为了遵循ISP，你常常需要先运用SRP来厘清职责。 |
