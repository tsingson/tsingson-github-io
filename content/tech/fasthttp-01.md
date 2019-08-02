---
title: "fasthttp系列文章(01)"
date: 2019-08-02T02:02:57+08:00
hidden: false
draft: false
author: "tsingson"

categories : [ "Development" ]
series: "fasthttp"
tags: [programming, golang]
keywords: [tsingson,gdihf,music,harmonica,blues]

description: "fasthttp系列文章"
slug: "fasthttp01"
---

![photo-desk](/tech/assets/photo-desk.jpg)



> fasthttp 文章系列:
> *  fasthttp web client 客户端(本文)
> *  fasthttp web server 服务端(及路由)
> *  fasthttp web server 中间件( 简单认证/ uber-go的zap日志/session会话...)
> *  fasthttp RESTful (处理JSON数据)
> *  fasthttp 处理 JWT (及 JWT安全性)
> *  fasthttp 对接非标准 web client (作为AAA, 数据加解密)
> *  fasthttp 部署


[简述]  [github.com/valyala/fasthttp](https://github.com/valyala/fasthttp) 是 golang 中一个标志性的高性能 HTTP库, 主要用于 webserver 开发, 以及 web client / proxy 等. fasthttp 的高性能开发思路, 启发了很多开发者.
<!--more-->



fasthttp 自己的介绍如下:

> Fast HTTP package for Go. Tuned for high performance. Zero memory allocations in hot paths. Up to 10x faster than net/http
> 
> Fast HTTP implementation for Go.
> 
>  
> Currently fasthttp is successfully used by [VertaMedia](https://vertamedia.com/)
in a production serving up to 200K rps from more than 1.5M concurrent keep-alive
connections per physical server.


事实上, 这有点小夸张, 但在一定场景下经过优化部署, 确是有很高的性能. 

近3年来, fasthttp 被我用在几个重大项目(对我而言, 项目有多重大, 与收钱的多少成正比) 中, 这里, 就写一个小系列, 介绍 fasthttp 的实际使用与经验得失.

>  --------------------------------------
>
>  想直接看代码的朋友, 请访问 [我写的 fasthttp-example](https://github.com/tsingson/fasthttp-example)
> 
> ------------------------------------------

## 0. 关于 fasthttp 的优点介绍

以下文字来自 [傅小黑](https://my.oschina.net/fuxiaohei) 原创文章: [Go 开发 HTTP 的另一个选择 fasthttp](https://my.oschina.net/fuxiaohei/blog/753977) 写于2016/09/30 :

> fasthttp 是 Go 的一款不同于标准库 net/http 的 HTTP 实现。fasthttp 的性能可以达到标准库的 10 倍，说明他魔性的实现方式。主要的点在于四个方面：
>
> * net/http 的实现是一个连接新建一个 goroutine；fasthttp 是利用一个 worker 复用 goroutine，减轻 runtime 调度 goroutine 的压力
> * net/http 解析的请求数据很多放在 map[string]string(http.Header) 或 map[string][]string(http.Request.Form)，有不必要的 []byte 到 string 的转换，是可以规避的
> * net/http 解析 HTTP 请求每次生成新的 *http.Request 和 http.ResponseWriter; fasthttp 解析 HTTP 数据到 *fasthttp.RequestCtx，然后使用 sync.Pool 复用结构实例，减少对象的数量
> * fasthttp 会延迟解析 HTTP 请求中的数据，尤其是 Body 部分。这样节省了很多不直接操作 Body 的情况的消耗
>
> 但是因为 fasthttp 的实现与标准库差距较大，所以 API 的设计完全不同。使用时既需要理解 HTTP 的处理过程，又需要注意和标准库的差别。

这段文字非常精练的总结了 fasthttp 的特点, 我摘录了这部分放在这里, 感谢 [傅小黑](https://my.oschina.net/fuxiaohei)  --- 另外, [傅小黑](https://my.oschina.net/fuxiaohei) 的技术文章非常棒, 欢迎大家去围观他....


## 1. 从 HTTP 1.x 协议说起

## 2. fasthttp 非"标准"的争议, 及为什么选择fasthttp

## 3. fasthttp 中的两种类型web client 

## 4. fasthttp 客户端实践

## 5. 小结


_

### 关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  


_

_
 [tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://tsingson.github.io/music/about-studio/),   2019/08/02





