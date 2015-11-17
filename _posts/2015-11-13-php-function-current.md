---
layout: post
title: php中foreach与current探究
date: 2015-11-13
categories: php
tags: [php, foreach, current]
description: 意外php中current的奇怪输出，探究一番
---

##引子
最近发现了一个问题关于foreach与current的问题，直接看例子：<br>

###Q1:

    <?php
        $arr = range(1, 3);
        var_dump(current($arr));
        foreach($arr as $val) {
            var_dump(current($arr));
        }
        var_dump(current($arr));

这段代码会得到这个结果：<br>

    int(1)
    int(2)
    int(2)
    int(2)
    int(2)

那么问题来了，手册上说current不会改变指针指向，为什么之后的`var_dump(current($arr))`都是输出`int(2)`？<br>

###Q2:

    <?php
        $arr = range(1, 3);
        var_dump(current($arr));
        foreach($arr as $val) {
        }
        var_dump(current($arr));

这段代码的结果是：<br>

int(1)
bool(false)

为什么我执行一次空的foreach，current会变成`false`，他现在到底指向了哪里？<br>

##探究

我将这个问题提交到stackoverflow [php: Difficult to understand function current](http://stackoverflow.com/questions/33685018/php-difficult-to-understand-function-current?noredirect=1),在回复中，有人说可能是受php版本影响，[https://3v4l.org/4iJj8](https://3v4l.org/4iJj8)，发现的确在php7中问题得到了修复。<br>

由于这个问题是在看[鸟哥](http://www.laruence.com/)的博客中想起来的，于是在twitter上@了并提问`是否是bug`，然而还没有收到答复。<br>

之后想到php.net上提交个bug单<br>
![](http://8.shikun.wang/img/php-bug.png)

既然说我提的bug重了，那就看看吧。果然，在其中一个单里找到了相同的问题[Bug #53405 	accessing the iterator inside a foreach loop leads to strange results](https://bugs.php.net/bug.php?id=53405&edit=2)<br>

发现最后一条回复：
`    [2015-03-09 11:16 UTC] nikic@php.net

    This was fixed by [https://wiki.php.net/rfc/php7_foreach](https://wiki.php.net/rfc/php7_foreach), the output is now as expected.`<br>

这里大概讲的是之前版本foreach的边界没有做好处理，导致产生了这个问题。<br>

##小结
既然这是一个bug，那么在php7以前的版本中就不要使用这样的写法了<br>

##源码分析
这个之后看看<br>


##参考资源
[Bug #53405 	accessing the iterator inside a foreach loop leads to strange results](https://bugs.php.net/bug.php?id=53405&edit=2)<br>
[PHP RFC: Fix "foreach" behavior](https://wiki.php.net/rfc/php7_foreach)<br>
[风雪之隅](http://www.laruence.com/)<br>
[3V4L](https://3v4l.org/4iJj8)<br>
[php: Difficult to understand function current](http://stackoverflow.com/questions/33685018/php-difficult-to-understand-function-current?noredirect=1)<br>
[深入解析php中的foreach问题](http://www.jb51.net/article/39299.htm)<br>

