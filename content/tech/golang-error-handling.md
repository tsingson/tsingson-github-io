---
title: "go语言的 error handling 是不是一个败笔"
date: 2019-08-03T02:02:57+08:00
hidden: false
draft: false
author: "tsingson"

categories : [ "Development" ]
series: "golang"
tags: [programming, golang]
keywords: [tsingson,gdihf,music,harmonica,blues]

description: "go语言的 error handling 是不是一个败笔"
slug: "golang-error-handling"

---

![photo-desk](/tech/assets/photo-desk.jpg)



go语言的 error handling 是不是一个败笔?  这是知乎上的一个提问, 我写了一些看法: 过于简单, 但不算败笔.

<!--more-->



## 0. 概述

利益相关: 主要用 golang 的自由职业者. 2014年学go 用了3个月, 后从 python/java 转用 go 为主, 中型委托开发项目做过3个 ( postgres + go + haskell ) 



Error Handling 从两方面来谈

1. 语言层面
2. 业务开发层面



### ## 1. 语言层面

没错, golang 的 error handling 过于简单, 并且是一个可以自定义的 interface , 很多时候, 有些"强制" 开发者在处理业务逻辑过程, 在需要处理错误的地方, 不能采用开发者"认为"语言本身应该携带的上下文与调用stack来识别错误并做对应处理

从这一点来说, 有点"失败", 但也基本能用. 我想, 这也是 go 2 正在寻找解决方案的原因吧.

同时, go 的哲学, 是代码与编译实现, 都强调"可读性", 也就是说, 代码看起什么样, 运行起来也是什么样, 很少语法糖.

从 go 最近被否掉的 try , 所以讨论中, 可以看出, go 核心团队对 try 像一个语法糖, 而不是一个直面问题核心的方案而诸多讨论.

再者, go 语言的发展, 非常非常克制, 并且非常克守"后向兼容"的承诺, 希望 go 2 可以直接编译运行 go 1 的代码并工作良好. 在 go 2 规划中, 有把一些标准库移出 go 语言核心的计划, 说白了就是语言核心是稳定的, 而标准库放在外面维护, 可以对标准库进行增强/重构甚至在 go module 语义版本支持下扩展新 api 或新功能. 这样, go 1.0 与 go 2 甚至 go 3 都各自能编译并运行良好. 

这是一个非常有趣的愿景, 而 go 核心团队正非常克制的实施中.

从 go 的 module 这事事件, 很能反映出 go 语言核心团队的一些思路.

所以, ***go 1 的 error handling 不算败笔, 但过于简单( 能用)***



### ## 2. 业务开发层面

在具体业务开发代码时, 个人是选择了一些 go 语言核心之外扩展的 error 库, 再加上自己的定制, 足够用也相对简单, 开发效率不低

从 golang 的一些杀手级项目, 如 docker , k8s 的源码来看, 扩展后的 error handling 还行, 工作得秒错.



### ## 3. 那么被认为是败笔?

那为什么 go 1 的 error handling 被吐槽, 我想, 可能是:

1. go 1 的 error handling 虽然能用, 但真的太简单了, 同时没有语法糖, 满屏的 if err != ....... 看起来确是不符合某些程序的"完美", "优雅"的内心
2. go 即克制又年轻, 10年, 在语言层面, 好像没变化, 对于使用过其他开发语言的朋友们来说, 对这个简单而不"优雅"的 error handling 积怨已久
3. 从 go module 事件来看, go 核心团队固执又封闭, ( 当然, 最近两年好了一些, go 团队公开的沟通多了不少) , 让不少朋友怨点不少



### ## 4. 开发建议

对自己的代码, 封装自己喜欢用的 error handling, 比如, 加上 error type, error code , error message 再分别处理.

对其他开源代码, 在开源代码上封装一层, 识别对方的 error 后, 加上自己的 error interface





以上, 仅供参考

祝码钱愉快...