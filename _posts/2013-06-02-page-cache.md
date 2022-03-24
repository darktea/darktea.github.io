---
title: Linux Page Cache and Buffer Cache
date: 2013-06-02 00:00:00 +0800
categories: [notes]
tags: [linux]
---

# 〇, 目的

Linux Kernel 为存取速度慢的 Block 设备提供了两种比较通用的 Cache 机制:

* Page Cache: 为以页 (Page) 为单位的操作提供 Cache
* Buffer Cache: 为以块 (Block) 为单位的操作提供 Cache

本文的目的就是介绍这两种 Cache 的相关知识.

(待)
