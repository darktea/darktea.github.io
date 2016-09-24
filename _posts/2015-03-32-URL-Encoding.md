---
layout: post
title: URL Encoding
category: notes
---

# 一, 要解决的问题

给一段 Url 参数中编码过（可能用的是 GBK 编码，也可能用的是 UTF-8 编码）的中文串，

在不知道是哪种编码时（可能是 GBK，也可能是 UTF-8），要都能成功解出中文

例如对于
```
%C1%B9%D0%AC%C5%AE
```
和
```
%E5%87%89%E9%9E%8B%E5%A5%B3
```

都要能解出中文：**凉鞋女**

**本文的组织方式：**

* 背景知识
 * URL 的语法和编码
 * 字符集及字符编码
* 解决方案

# 二, 背景知识

在讲 URL 的 ecoding 之前，需要先了解 URL 的语法。

## 1. URL 组成

URL 一般由几个部分组成。例如：

```
https://bob:bobby@www.lunatech.com:8080/file;p=1?q=2#third
```

|Part|Data|
| ------------- | ------------- |
|Scheme|https|
|User|bob|
|Password|bobby|
|Host address|www.lunatech.com|
|Port|8080|
|Path|/file|
|Path parameters|p=1|
|Query parameters|q=2|
|Fragment|third|

## 2. URL 的保留字

从 URL 的组成也可以看出, 有一些字符是 URL 的保留字。

但有需要特别注意：有些字符对 URL 所有的部分都是保留字；但有些字符只是对 URL 的某个部分是保留字。

* "?" 对 Query 部分不需要转义
* "/" 对 Query 部分不需要转义
* "=" 对 Path 部分不需要转义; 对 Query 的 值的部分不需要转义
* 对 Path 部分不需要转义：

```
  ":@-._~!$&'()*+,;="
```

* 对 Fragment 部分不需要转义：

```
"/?:@-._~!$&'()*+,;="
```


## 3. URL 保留字编码

既然 URL 语法中存在保留字，那么当需要真正使用到这些保留字的时候，就需要对这些保留字进行编码（也就是转义），例如：英文问号（?）转为 %3F （％百分号开头，随后是16进制的数字）。

但就像前面提到的，URL 各个部分的保留字不同；所以，各个部分的 encoding 规则也不同。例如：

* 对 PATH 部分，空格被编码为：％20；而英文加号（+）不需要编码
* 对 Query 部分，空格可以被编码为：英文加号（+）或者 %20；而英文加号（+）必须被编码为：%2B

Query 部分的例子：

blue+light blue

应该被编码为：blue%2Blight+blue

**结论**：

对 URL 转义的时候，不能对整个 URL 进行处理；而是需要分别对 URL 各个部分做处理。

**这里我们只关注请求参数的编码请求参数中：**

* 空格可以被编码为：英文加号（+）
* 空格也可以被编码为：%20
* 而英文加号（+）必须被编码为：%2B

## 4. 非 ASCII 字符的 URL 编码

根据 URL 规范，将非 ASCII 字符按照某种 **编码格式** 编码成 16 进制数字然后将每个 16 进制表示的字节前加上“%”

这里所说的 **编码格式** 对我们来说就是两种编码 GBK，UTF-8 中的一种。

## 5. 例子

凉鞋女：

* GBK编码：C1B9  D0AC C5AE （每个字2个Bytes，一共6个Bytes）
* UTF编码： E58789 E99E8B E5A5B3 （每个字3个Bytes，一共9个Bytes）

编码工具：

* http://r12a.github.io/apps/conversion/

所以其 URL 编码分别为（每Byte前加一个百分号）：

* %C1%B9%D0%AC%C5%AE
* %E5%87%89%E9%9E%8B%E5%A5%B3

# References:

* [What every web developer must know about URL encoding](http://www.oschina.net/translate/what-every-web-developer-must-know-about-url-encoding)