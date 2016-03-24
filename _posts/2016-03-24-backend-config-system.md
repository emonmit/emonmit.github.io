---
layout: post
title: 也许你真的需要一个配置管理系统
date: 2016-03-24
categories: 系统设计 工作效率提升
tags: [系统设计,工作效率提升]
description: 忍受不了产品频繁修改需求，那就给他们一个自己修改的空间；减少了发版次数又让自己更专注的进行开发，何乐而不为。
---
随意转载，请注明出处：[http://8.shikun.wang/php/2016/03/24/backend-config-system/](http://8.shikun.wang/php/2016/03/24/backend-config-system/)
[TOC]
##源起
有一次在做公司的一个微信活动，之后发现对于活动的起止时间特别不好控制（需求本身在变），每次都需要修改代码，重新发代码（当时没经验，太相信产品……），这在代码管理上是不能忍的。而且这活动也是老大帮其他组做的，可能一年只有这一个活动，也不好做一张活动管理的表，毕竟只有一条数据。
<br />
另一方面，在日常的开发中，产品也经常对一些文案性的东西进行修改，每次修改完还急着说，马上发版！很急的！只能无奈的让他们检查下还有没有遗漏的地方，然后给他们发版，目前代码已经200+M，有的时候一天就发1G，而且只有那么几个字的变更，这样一算成本就太大了。
<br />
就这样，我在一次迭代会上提出了做一个后台配置管理模块的建议，然后这事也落到了自己头上，算不算给自己挖了个坑，不过为了以后能偷懒，这个坑踩就踩了。
<br />
其实这个配置管理模块还有个**最大**的好处，就是节省后台开发成本，有时候一个需求就配套一个管理后台，而这个后台的作用仅仅是配那么寥寥几条配置信息。为此，就需要一套增删改查的代码，虽说熟练了做一套这后台也不用多久，但真的觉得不美了！不就是要配几条数据吗，全集中给我，我把接口给你，一行代码搞定原来几百行的功能不好吗。事实证明，的确很成功。
<br />
##设计
`感谢可爱的组员们一起跟我讨论、提建议、做评审、尝试使用，在大家一次次的检验中，才让这个模块不是形同虚设。`<br />

首先这个模块的定位是解决开发中碰到的**开发成本大但功能简单的配置**管理问题。预期最终效果是集中管理所有业务的简单配置项。之所以定位是`简单配置项`，是因为一些后台配置与业务有着非常紧的联系，单一的简单配置功能无法满足业务上的要求，无法很好的分离出来。虽说做也是能做出来，但如果真的做出来了，那么他不应该叫配置管理模块，而应该是后台管理系统……
<br />
###第一版：设计之美、使用之殇
####什么是“配置”？
首先是对配置的抽象，由于是要应用到各个业务中，这个抽象尤为重要。<br />
一个配置要使用，首先是一个key->value，我们把key传给接口，来取得我们想要的配置。<br />
接着新的问题就来了，可能有的配置会在各个业务端都有使用。比如说微信分享中：`shareTitle/shareDesc/link/imgUrl`，用这样一个命名显然无法区分了。所以还需要和业务结合起来。而本身公司的业务基本是一个树形的结构，所以想到用树的结构将这些配置集合起来，所以，在key/value/valueType之后，加上id/pid/isLeaf等几个字段，将这些配置集合成一个树形结构，通过id来取值。<br />
####洪水猛兽
提出用树形结构之后，分分钟大家就说用树的各种NB各种上天了。总结一下……<br />
- 理论上无限深的层级（光这一点，基本满足所有配置需求）
- 将整颗树缓存到redis，提升读性能
<br />
虽然最后含着泪，但我还是笑着做完了这一版，完成的点主要有：<br />
- 嗯...理论上无限深的层级，但valueType只支持单行的文本类型
- 拖动排序、更换分支（黑科技）
- 定期生成完整树缓存(redis)，也可手动更新树枝缓存（任意分支）
<br />
####GG
嗯，虽然做完了大家都感觉狂拽炫酷X炸天，但最后还是毅然决然的放弃了这一版，因为整个模块都很好懂，`配置管理树\父节点\子节点\叶子节点……`怕产品运营玩不来。另外，这样的设计在数据膨胀的时候优化起来也并不那么容易。
<br />

###第二版：精简设计、好评如潮
经历了第一版的过度设计，第二版的针对性更强了，整体的思路与第一版一样。但是层级上首先限死了只有4层：`业务端`,`大业务模块`,`子业务模块`,`具体配置项`。在一定程度上缩小了可配置范围，只用于处理业务耦合度低的配置项。
<br />

由于定位是处理简单的业务耦合度低的配置项，首先，对可配置类型就限制在了简单的：<br />
- 单行文本 （几个字、一段话、一个id、各种值……）
- 自动索引多行文本 （同类的多条文本，只需要有排序）
- 手动索引多行文本 （比如一个产品的多种属性，无排序关系却要区分不同的属性值）
<br />
尽管只有这几种配置，已经能解决大部分碰到的配置管理问题了，也算是没有违背初衷。
<br />
相比于第一版，第二版简易了很多，数据也都直接存在数据库中，也可以提供很可靠的服务。好事多磨，最终效果还是很满意的。

####最终表现
<br />
就不po图了，简单描述下：<br />
业务端<br />
- 各端代码直接引用配置管理工具类即可
- 由于线上线下数据库不同，没有用id作为key，而是用完整的业务分支路径和配置项key组合作为真正key
<br />
管理后台<br />
- 新增大业务模块只需要在控制器中加入一小段代码（复制粘贴、改个actionName）
- 所有配置只需要在后台添加，就能完成配置，不需要在写后台模块
<br />
也就是说，在以前，拿到一个后台需求之后，可能要新增module、新建表、写一套增删该查、数据校验、拖动排序、前端优化等等，起码需要一个下午的时间。但是用上了这样一套后台配置管理系统的话，只需要10分钟，就能完成线上线下包括前端展示上的调试。<br />
##结语
之后，我把很多需要配置的、或者产品让我修改了2次以上的东西都放在了配置管理里面，让他们自己去改。<br />
总的来说，达到甚至超越了预期的效果，最明显的是降低了RD的被打断次数、发版次数，并且开发后台的时间节省了非常多。
<br />

**如果你所在的团队还没有这样的一套系统，那么行动起来吧，提升工作效率利器！**