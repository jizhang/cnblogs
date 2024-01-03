---
layout: post
title: "MySQL 异常 UTF-8 字符的处理"
date: 2014-10-14 13:16
comments: true
categories: [Programming]
tags: [mysql, etl]
published: true
---

ETL流程中，我们会将Hive中的数据导入MySQL——先用Hive命令行将数据保存为文本文件，然后用MySQL的LOAD DATA语句进行加载。最近有一张表在加载到MySQL时会报以下错误：

```
Incorrect string value: '\xF0\x9D\x8C\x86' for column ...
```

经查，这个字段中保存的是用户聊天记录，因此会有一些表情符号。这些符号在UTF-8编码下需要使用4个字节来记录，而MySQL中的utf8编码只支持3个字节，因此无法导入。

根据UTF-8的编码规范，3个字节支持的Unicode字符范围是U+0000–U+FFFF，因此可以在Hive中对数据做一下清洗：

```sql
SELECT REGEXP_REPLACE(content, '[^\\u0000-\\uFFFF]', '') FROM ...
```

这样就能排除那些需要使用3个以上字节来记录的字符了，从而成功导入MySQL。

以下是一些详细说明和参考资料。

<!-- more -->

## Unicode字符集和UTF编码

[Unicode字符集](http://en.wikipedia.org/wiki/Unicode)是一种将全球所有文字都囊括在内的字符集，从而实现跨语言、跨平台的文字信息交换。它由[基本多语平面（BMP）](http://en.wikipedia.org/wiki/Plane_\(Unicode\)#Basic_Multilingual_Plane)和多个扩展平面（non-BMP）组成。前者的编码范围是U+0000-U+FFFF，包括了绝大多数现代语言文字，因此最为常用。

[UTF](http://en.wikipedia.org/wiki/Unicode#Unicode_Transformation_Format_and_Universal_Character_Set)则是一种编码格式，负责将Unicode字符对应的编号转换为计算机可以识别的二进制数据，进行保存和读取。

比如，磁盘上记录了以下二进制数据：

```
1101000 1100101 1101100 1101100 1101111
```

读取它的程序知道这是以UTF-8编码保存的字符串，因此将其解析为以下编号：

```
104 101 108 108 111
```

又因为UTF-8编码对应的字符集是Unicode，所以上面这五个编号对应的字符便是“hello”。

很多人会将Unicode和UTF混淆，但两者并不具可比性，它们完成的功能是不同的。

## UTF-8编码

UTF编码家族也有很多成员，其中[UTF-8](http://en.wikipedia.org/wiki/UTF-8)最为常用。它是一种变长的编码格式，对于ASCII码中的字符使用1个字节进行编码，对于中文等则使用3个字节。这样做的优点是在存储西方语言文字时不会造成空间浪费，不像UTF-16和UTF-32，分别使用两个字节和四个字节对所有字符进行编码。

UTF-8编码的字节数上限并不是3个。对于U+0000-U+FFFF范围内的字符，使用3个字节可以表示完全；对于non-BMP中的字符，则会使用4-6个字节来表示。同样，UTF-16编码也会使用四个字节来表示non-BMP中的字符。

## MySQL的UTF-8编码

根据MySQL的[官方文档](http://dev.mysql.com/doc/refman/5.5/en/charset-unicode.html)，它的UTF-8编码支持是不完全的，最多使用3个字符，这也是导入数据时报错的原因。

MySQL5.5开始支持utf8mb4编码，至多使用4个字节，因此能包含到non-BMP字符。只是我们的MySQL版本仍是5.1，因此选择丢弃这些字符。

## 参考资料

* http://stackoverflow.com/questions/3951722/whats-the-difference-between-unicode-and-utf8
* http://www.joelonsoftware.com/articles/Unicode.html
* http://apps.timwhitlock.info/emoji/tables/unicode
