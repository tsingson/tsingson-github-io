---
title: "goim 客户端之消息发送/接收"
date: 2019-05-24T12:02:57+08:00
hidden: false
draft: true
author: "tsingson"
categories : [ "Development" ]
series: "goim"
tags: [goim,golang]
keywords: [tsingson]

description: "goim 客户端之消息发送/接收实现解读与定制"
slug: "goim-go-05"
---

----


> goim 文章系列(共5篇):
> * [goim 架构与定制](/tech/goim-go-01/index.html)
> * [从goim定制, 浅谈 golang 的 interface 解耦合与gRPC](/tech/goim-go-02/index.html)
> * [goim中的 bilibili/discovery (eureka)基本概念及应用](/tech/goim-go-03/index.html)
> * [goim 的 data flow 数据流](/tech/goim-go-04/index.html)
> * [goim的业务集成(分享会小结与QA)](/tech/goim-go-06/index.html)
>
>  有个 slack 频道, 不少朋友在交流 goim , 欢迎加入[slack #goim](https://join.slack.com/t/reading-go/shared_invite/enQtMjgwNTU5MTE5NjgxLTA5NDQwYzE4NGNhNDI3N2E0ZmYwOGM2MWNjMDUyNjczY2I0OThiNzA5ZTk0MTc1MGYyYzk0NTA0MjM4OTZhYWE)






## 0. 文章撰写动机

goim 官网 [http://goim.io](http://goim.io)

goim 源码 [https://github.com/Terry-Mao/goim](https://github.com/Terry-Mao/goim)

___

写了几个 goim 相关小文章后, 有点感慨:
> goim 的文章已经很多了.
> 我写了 goim 文章后就有点后悔了: 
>
> 看到比较早的文章, 是 2016年,[goim源码剖析](https://laohanlinux.github.io/2016/12/22/goim源码剖析/)
>
> 很多文章已经把 goim 说明得很清楚很明白了, 各文章作者关注点不同(版本也不同) , 有些文章非常非常好.

> 从 goim 询问到的问题来看, 就两大类:

> 1. 阅读代码过程, 有些没看懂原始设计意图, 或对某些局部代码的使用不太清楚
> 2. 如何实现一些业务, 包括鉴权认证啦, 消息合并啦, 离线存储或聊天机器人如何写啦, 如何部署及压测啦....

> 我只能这样建议, 阅读代码, 阅读代码, 阅读代码, 代码说得比较清楚. ( 相比其他 im 开源库, 这是我欣赏 goim 的地方, 代码背后的设计意图一脉相承而又保持足够简单, 让 goim 成为一个优秀的 im 实现code base , 易于扩展/维护/集成...)

> 很开心的是, 因 goim 认识更多有趣的朋友, 谢谢!

----

网上很少有 goim 客户端的相关文章, 毕竟, goim 的下行客户端有 websocket javascript / java 实现例子, 上行有 http API 以及 golang 的例子.

这里 [https://github.com/Terry-Mao/goim/issues/298](https://github.com/Terry-Mao/goim/issues/298) 有个issue 讨论, 还是挺有意思

所以, 我想, 这篇小文写写这个 goim 客户端相关的:
1. 解读一下 goim 客户端的 protocol 及 api 定义(部分只有定义, 没有相应实现代码)
2. 讨论一下 goim 的消息发布与接收相关方法
3. 讨论一下网上谈到的 goim 消息发送与接收合一的扩展



## 1. goim 客户端相关的协议
### 1.1 数据封装( 序列化与反序列化)

> Serialization and Deserialization 序列化与反序列化, 这是所有程序员即熟悉又最经常处理的一类数据操作
>
> **序列化**：把对象转换为字节序列的过程称为对象的序列化。
>
> **反序列化**：把字节序列恢复为对象的过程称为对象的反序列化。

以下引用自 [wiki](https://en.wikipedia.org/wiki/Serialization)

> In [computer science](https://en.wikipedia.org/wiki/Computer_science), in the context of data storage, **serialization** (or serialisation) is the process of translating [data structures](https://en.wikipedia.org/wiki/Data_structure) or [object](https://en.wikipedia.org/wiki/Object_(computer_science)) state into a format that can be stored (for example, in a [file](https://en.wikipedia.org/wiki/Computer_file) or memory [buffer](https://en.wikipedia.org/wiki/Data_buffer)) or transmitted (for example, across a [network](https://en.wikipedia.org/wiki/Computer_network) connection link) and reconstructed later (possibly in a different computer environment).[[1\]](https://en.wikipedia.org/wiki/Serialization#cite_note-1) When the resulting series of bits is reread according to the serialization format, it can be used to create a semantically identical clone of the original object. For many complex objects, such as those that make extensive use of [references](https://en.wikipedia.org/wiki/Reference_(computer_science)), this process is not straightforward. Serialization of object-oriented [objects](https://en.wikipedia.org/wiki/Object_(computer_science)) does not include any of their associated [methods](https://en.wikipedia.org/wiki/Method_(computer_science)) with which they were previously linked.
>
> This process of serializing an object is also called [marshalling](https://en.wikipedia.org/wiki/Marshalling_(computer_science)) an object.[[2\]](https://en.wikipedia.org/wiki/Serialization#cite_note-2) The opposite operation, extracting a data structure from a series of bytes, is **deserialization** (also called **unmarshalling**).

以下是[中文wiki](https://zh.wikipedia.org/wiki/序列化)

> **序列化**（serialization）在[計算機科學](https://zh.wikipedia.org/wiki/計算機科學)的資料處理中，是指將[資料結構](https://zh.wikipedia.org/wiki/資料結構)或物件狀態轉換成可取用格式（例如存成檔案，存於緩衝，或經由網絡中傳送），以留待後續在相同或另一台計算機環境中，能恢復原先狀態的過程。依照序列化格式重新獲取位元組的結果時，可以利用它來產生與原始物件相同語義的副本。對於許多物件，像是使用大量參照的複雜物件，這種序列化重建的過程並不容易。物件導向中的物件序列化，並不概括之前原始物件所關聯的函式。這種過程也稱為**物件編組 （marshalling）**。從一系列位元組提取資料結構的反向操作，是**反序列化（也稱為解編組、deserialization、unmarshalling）**。



从上面的描述, 简单来说, 序列化与反序列化就是一个互逆的过程, 可以相互验证

## 1.2  protocol 协议 
简单来说, protocol 协议 就是一个约定, 一个共同遵守的规则

## 2  goim 的 TCP 客户端编写示例
### 2.1 


