---
date: '2025-09-02T21:22:10+08:00'
author: "Eric"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: true
description: "花了一些时间折腾了一个翻译软件出来"
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
title: "trs"
categories: "开发"
tags: ["小东西","python"]
---

# trs

今天花了几个小时从零折腾一些库，开发了一个python翻译工具脚本，当读入单词时请求bing网页再用bs处理得到词的解释，当读入句子时请求大模型得到智能翻译，并使用rich库美化了输出。

[仓库地址](https://github.com/2667783575/trs)

并且从中学习到了python脚本如何转换为系统自动调用python运行的脚本

即在python脚本头添加shebang头，然后将python脚本的名称中的.py删去即可

## 二次开发

2025/9/3  11:43

在群友的建议下，晚上又花了一些时间在不影响原本功能的情况下添加了复制和保存功能
