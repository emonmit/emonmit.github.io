---
layout: post
title: php中foreach与current探究
date: 2015-11-13
categories: php
tags: [php, foreach, current]
description: foreach中current的奇怪输出，探究一番
---
随意转载，请注明出处：[http://8.shikun.wang/php/2015/11/13/php-function-current/](http://8.shikun.wang/php/2015/11/13/php-function-current/)

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

我将这两个问题提交到stackoverflow [php: Difficult to understand function current](http://stackoverflow.com/questions/33685018/php-difficult-to-understand-function-current?noredirect=1),在回复中，有人说可能是受php版本影响，[https://3v4l.org/4iJj8](https://3v4l.org/4iJj8)，发现的确在php7中问题得到了修复。<br>
![](http://emonmit.github.io/img/3v4l-foreach.png)

之后到php.net上提交个bug单<br>
![](http://emonmit.github.io/img/php-bug.png)

既然说我提的bug重了，那就看看吧。果然，在其中一个单里找到了跟`问题2`相同的问题[Bug #53405 	accessing the iterator inside a foreach loop leads to strange results](https://bugs.php.net/bug.php?id=53405&edit=2)<br>

看起来这个问题也经过一番论证，当然，最后还是定义为bug。<br>

引用`nikic`对补丁的说明，包括问题产生原因以及解决办法：<br>

    Currently there are two ways to iterate an array: Either using the internal
    array pointer or using an external HashPosition. Right now the latter isn't
    interruption safe though and can't be used in any iteration that runs user
    code.
    
    For that reason foreach had to use the IAP for the iteration. This created
    a bunch of issues: Firstly foreach is often required to copy the array it
    iterates even though it should not be strictly necessary according to COW.
    Secondly using the IAP created weird behavior of current() etc in the loop
    body, that was furthermore heavily dependent on just how exactly the looping
    was done. Thirdly the behavior when modifying the array during iteration
    is very unpredictable.
    
    This patch approaches the problem by making external HashPosition array
    iterators interruption safe. This is done by adding two new APIs:
    
    void zend_track_hash_position(HashTable *ht, HashPosition *pos);
    void zend_untrack_hash_position(HashTable *ht, HashPosition *pos);
    
    Using this functions the HashPosition has to be registered before the
    iteration and unregistered after it. If the HashPosition is registered
    in such a way the zend_hash operations will properly update the
    HashPosition pointer on modification, just like it is usually done for
    the IAP.

想要知道具体变更[点击这里](https://github.com/nikic/php-src/commit/185c7fae3a3d7a90693c24726a8f2685b3751464)<br>

在[修复文档](https://wiki.php.net/rfc/php7_foreach)中，将问题定性为`对foreach的一些边缘情况缺乏测试`;提到了更新后的foreach实现中，将`FE_RESET`、`FE_FETCH`这两个opcode分解成了`FE_RESET_R`、`FE_FETCH_R`、`FE_RESET_RW`、`FE_FETCH_RW`，后缀`_R`的应用于值传递时，`_RW`应用于引用时。更多的实现细节我也没看太懂，先放出原文吧，等熟悉了再更新：<br>

    Implementation Details
    
    The existing FE_RESET/FE_FETCH opcodes are split into separate FE_RESET_R/FE_FETCH_R opcodes used to implement foreach by value and FE_RESET_RW/FE_FETCH_RW to implement foreach by reference. The suffix _R means that we use array (or object) only for reading, and suffix _RW that we also may indirectly modify it. A new FE_FREE opcode is introduced. It's used at the end of foreach loops, instead of FREE opcode.
    
    Iteration by value over array doesn't use or modify internal array pointer. The value of the pointer is kept in reserved space of temporary variable used for iteration. It's acceptable through Z_FE_POS() macro.
    
    Iteration by reference or by value over plain object implemented using special HashTableIterator structures.
    
    typedef struct _HashTableIterator {
        HashTable    *ht;
        HashPosition  pos;
    } HashTableIterator;
    
    On entrance into foreach loop FE_RESET_R/RW opcode creates and initializes a new iterator and stores its index in reserved space of temporary variable used for iteration. On exit, FE_FREE opcode removes corresponding iterator.
    
    Iterators are actually allocated in a buffer - EG(ht_iterators), represented by plain array. The more nested foreach by reference iterators the bigger buffer we will need. We start with small preallocated buffer - EG(ht_iterators_slots), and then extend it if necessary in heap. EG(ht_iterators_count) keeps the number of available slots for iterators, EG(ht_iterators_used) - the number of used slots.
    
    struct _zend_executor_globals {
        ...
        uint32_t           ht_iterators_count;     /* number of allocatd slots */
        uint32_t           ht_iterators_used;      /* number of used slots */
        HashTableIterator *ht_iterators;
        HashTableIterator  ht_iterators_slots[16];
        ...
    }
    
    Creation, deletion and accessing iterators position is implemented through special API.
    
    ZEND_API uint32_t     zend_hash_iterator_add(HashTable *ht);
    ZEND_API HashPosition zend_hash_iterator_pos(uint32_t idx, HashTable *ht);
    ZEND_API void         zend_hash_iterator_del(uint32_t idx);
    
    Indirect modification of iterators positions implemented through zend_hash_iterators_update(). It's called when HashTable modification may affects iterator position. For example when element referred by iterator is inserted, or when iterator is set at the end of the array and new element is inserted.
    
    ZEND_API void         zend_hash_iterators_update(HashTable *ht, HashPosition from, HashPosition to);

    Foe more details see zend_hash_iterators_*() functions implementation in zend_hash.c

更多信息可查看:[PHP RFC: Fix "foreach" behavior](https://wiki.php.net/rfc/php7_foreach)<br>

##小结

既然这是一个bug，那么在php7以前的版本中就不要使用这样的写法了，以防产生其他的问题。<br>

##深度分析

`欢迎吐槽，毕竟尚未完全弄懂。以下分析以问题1为例。ps:还没对问题2分析`<br>

为了方便起见，我先将我代码中vld的信息拿出来：<br>

    Finding entry points
    Branch analysis from position: 0
    Jump found. Position 1 = 9, Position 2 = 17
    Branch analysis from position: 9
    Jump found. Position 1 = 10, Position 2 = 17
    Branch analysis from position: 10
    Jump found. Position 1 = 9
    Branch analysis from position: 9
    Branch analysis from position: 17
    Jump found. Position 1 = -2
    Branch analysis from position: 17
    filename:       /in/RL3TZ
    function name:  (null)
    number of ops:  23
    compiled vars:  !0 = $arr, !1 = $val
    line     #* E I O op                           fetch          ext  return  operands
    -------------------------------------------------------------------------------------
       2     0  E >   SEND_VAL                                                 1
             1        SEND_VAL                                                 3
             2        DO_FCALL                                      2  $0      'range'
             3        ASSIGN                                                   !0, $0
       3     4        SEND_REF                                                 !0
             5        DO_FCALL                                      1  $2      'current'
             6        SEND_VAR_NO_REF                               6          $2
             7        DO_FCALL                                      1          'var_dump'
       4     8      > FE_RESET                                         $4      !0, ->17
             9    > > FE_FETCH                                         $5      $4, ->17
            10    >   OP_DATA                                                  
            11        ASSIGN                                                   !1, $5
       5    12        SEND_REF                                                 !0
            13        DO_FCALL                                      1  $7      'current'
            14        SEND_VAR_NO_REF                               6          $7
            15        DO_FCALL                                      1          'var_dump'
       6    16      > JMP                                                      ->9
            17    >   SWITCH_FREE                                              $4
       7    18        SEND_REF                                                 !0
            19        DO_FCALL                                      1  $9      'current'
            20        SEND_VAR_NO_REF                               6          $9
            21        DO_FCALL                                      1          'var_dump'
            22      > RETURN                                                   1
    
    Generated using Vulcan Logic Dumper, using php 5.6.0

根据[TIPI项目](www.php-internals.com/book/?p=chapt16/16-01-01-php-foreach&ref=chm&v=TIPI_2014-04-29_V0.8.3)中对foreach的分析：<br>

    源代码：
        $arr = array(1,2,3,4,5);
         
        foreach($arr as $key => $row) {
            echo key($arr), '=>', current($arr), "\r\n";
        }
    
    问题：为什么foreach循环体中执行key或current会显示第二个元素（非引用情况）？以key函数为例，我们执行函数调用时，会执行中间代码SEND_REF，此中间代码会将没有设置引用的变量复制一份并设置为引用。当进入循环体时，PHP内核已经经过了一次fetch操作，相当于执行了一次next操作，当前元素指向第二个元素。因此我们在foreach的循环体中执行key函数时，key中调用的数组变量为PHP执行了一次fetch操作的数组拷贝，此时foreach的内部指针指向第二个元素。

按这里的解释，中间代码第一次进行SEND_REF时，会将变量复制一份，而复制这个变量时，已经有过FE_FETCH操作了，所以current变成了第二个元素。那么再回头看我的代码，vld中第4行在foreach之前，已经产生了`!0`，并不是第一次进行SEND_REF，那结果为什么还是一样的呢，所以，我觉得这个说法并`不靠谱`。<br>

扩充：<br>
- [深入解析php中的foreach问题](http://www.3lian.com/edu/2013/07-01/77698.html#)`参考问题2，不知道是不是原出处，都不写转载源差评`
- [深入理解PHP原理之foreach](http://www.laruence.com/2008/11/20/630.html)`这里结合源码对foreach进行了详细讲解，orz鸟哥`

##参考资源
[Bug #53405 	accessing the iterator inside a foreach loop leads to strange results](https://bugs.php.net/bug.php?id=53405&edit=2)<br>
[PHP RFC: Fix "foreach" behavior](https://wiki.php.net/rfc/php7_foreach)<br>
[风雪之隅](http://www.laruence.com/)<br>
[3V4L](https://3v4l.org/4iJj8)<br>
[php: Difficult to understand function current](http://stackoverflow.com/questions/33685018/php-difficult-to-understand-function-current?noredirect=1)<br>
[深入解析php中的foreach问题](http://www.3lian.com/edu/2013/07-01/77698.html)<br>

