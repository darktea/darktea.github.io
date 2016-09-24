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

* URL 的语法和编码
* 字符集及字符编码
* 问题解决方案

# 二, URL 的语法和编码

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
:@-._~!$&'()*+,;=
```

* 对 Fragment 部分不需要转义：

```
/?:@-._~!$&'()*+,;=
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

# 三, 字符集及字符集编码

## 1. 字符集

* ASCII： 7bits 表示一个字符，共128字符
* ASCII的增强：
 * 8bits表示一个字符，共256字符。例如： ISO­8859­1 （西欧字符）
 * 2Bytes表示一个字符，GBK。（其特点后面详细讲）
* Unicode字符集：对全世界的每个字符，规定一个唯一的数字（其范围目前是U+0000~U+10FFFF，大概100万，32bits可以搞定 ）来代表。例如：\u5973 女
* 其他很多扩展字符集。。。

## 2. 字符集编码

字符集编码：计算机里面怎么存储、传输Unicode字符。例如：UTF-8，UTF-16，UTF-32 。。。

## 3. GBK

既是一种字符集又是一种编码。

* GB2312（双字节）：小于127的字符兼容ASCII，但两个大于127的字符连在一起时，就表示一个汉字，前面的一个字节（高字节）从0xA1到 0xF7，后面一个字节（低字节）从0xA1到0xFE。一共 7000 多个字符 （其中6千多个简体字）
* GBK （双字节）：微软对GB2312 进行扩展，加入繁体字，日语及朝鲜语汉字等。2万多个汉字
* GB18030 （四字节）：是中华人民共和国现时最新的内码字集，与GB2312完全兼容，与GBK基本兼容，支持GB 13000及Unicode的全部统一汉字

## 4. Unicode编码

Unicode 字符集目前的范围是 U+0000~U+10FFFF。对 Unicode 字符集有多种编码。

* UTF-32：直接用4字节存储 （有大小端问题）
* UTF-16：最早的时候，Unicode的范围没有这么大，可以直接用2字节搞定；现在2字节不够，UTF-16的编码做了更新，2字节或者4字节。 （同样有大小端问题）
* UTF-8：变长，最多4字节。没有大小端问题。可以检测到每个字符的边界。

## 5. Java 中的字符集编码

String使用 UTF-16（2字节或者4字节）：

```
String chinese="ab中文”
String b= new String(bs,“GBK"); // bs is a byte array
```

Refer:

* http://www.zhihu.com/question/27562173
* http://lukejin.iteye.com/blog/586088

# 四, 问题解决方案

* %E5%87%89%E9%9E%8B%E5%A5%B3 => 一个 java byte 数组
* Java byte 数组 => Java 字符串类型, 分别用 UTF-8 和 GBK 解码

```java
String a= new String(bs,“UTF-8");
String b= new String(bs,“GBK");
```

* 利用 nio 对 a 和 b 进行检测，判断是否是中文（没有乱码）：
 * java.nio.charset.Charset.forName("GBK").newEncoder().canEncode(a));
* 如果都返回 true（看起来都不是乱码）。用正则判断是否是 UTF-8，否则就是 GBK
 * 例外 case（正则判断是 UTF-8，但实际上还是 GBK）： "鏈條", "瑷媄", "妤媞", "浜叉鼎"

# 五, References:

* [What every web developer must know about URL encoding](http://www.oschina.net/translate/what-every-web-developer-must-know-about-url-encoding)