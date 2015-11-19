---
layout: post
title: php7编程规范 - 译文
date: 2015-11-19
categories: php
tags: [php, coding standards]
description: php7源码中编程规范文档翻译
---

##PHP编码规范

`更新中`

　　本文档列出了一些所有程序员在新编写或者修改已有代码时都需要遵从的标准。这篇文档直到PHP v3.0开发后期才被添加进来，代码库也并没有完全遵从这个编码规范，但是正在朝标准化发展。直到现在进入PHP5之后，已经有很多代码按照规范重新编写过了。<br>

###代码实现

0.  在源文件和手册中记录下你的代码。[tm]<br>
1.  函数参数中带有指针的话，不应该在函数体内被释放。<br>比如说, `function int mail(char *to, char *from)`，其中`to/from`二者任一都不能被释放。<br>
