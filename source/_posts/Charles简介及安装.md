---
title: Charles简介及安装
date: 2020-02-06
tags:
  - 抓包
categories: 抓包
---

## Charles简介

Charles是目前流行的http抓包调试工具，Mac、Unix、Windows各个平台都支持，其功能强大到包括：

1. 支持SSL代理，可以截取分析SSL的请求
2. 支持流量控制。可以模拟慢速网络以及等待时间（latency）较长的请求。
3. 支持AJAX调试。可以自动将json或xml数据格式化，方便查看。
4. 支持AMF调试。可以将Flash Remoting 或 Flex Remoting信息格式化，方便查看。
5. 支持重发网络请求，方便后端调试。
6. 支持修改网络请求参数。
7. 支持网络请求的截获并动态修改。
8. 检查HTML，CSS和RSS内容是否符合W3C标准

<!-- more -->

**与fiddler对比，Charles的优势：**

- Charles 是一款全平台的抓包工具，支持windows、mac、linux三大平台，而fiddler仅支持windows版。
- Charles提供两种视图，一种是按访问的域名分类，另一种是按访问的时间排序，fiddler提供一种视图，就是按访问的时间排序。

## 2、Charles工作原理

Charles的工作原理很简单，本质是就是一个http抓包分析工具，在工作的时候需要先把charles设置成代理服务器，这样所有的网络请求都会经过charles了。

## 3、Charles下载安装

Charles的安装的可以去官网http://www.charlesproxy.com/download/进行下载安装。