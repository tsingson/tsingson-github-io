---
title: goim交流分享会后的小结与QA
date: 2019-05-30T23:02:57+08:00
hidden: false
draft: false
author: "tsingson"

categories : [ "Development" ]
series: "goim"
tags: [goim,golang]
keywords: [tsingson,gdihf,music,harmonica,blues]

description: "goim 交流分享会后的小结与QA"
slug: "goim-go-06"

---



 



![go-reading](/tech/assets/go-reading.jpg)




[简述]  [http://goim.io](http://goim.io) 是 非常成功的 IM (Instance Message) 即时消息平台 , 本文汇总在 [go夜读](https://github.com/developer-learning/reading-go) 组织的 goim 交流分享会后的小结 
<!--more-->



> goim 交流分享会的视频, 第一次用视频会议, 一堆问题, 请包容一下 :(
>
> 视频地址在 youtube [https://www.youtube.com/watch?v=bFniRH3ifx8](https://www.youtube.com/watch?v=bFniRH3ifx8) 

----

> goim 文章系列(共5篇):
> * [goim 架构与定制](/tech/goim-go-01/index.html)
> * [从goim定制, 浅谈 golang 的 interface 解耦合与gRPC](/tech/goim-go-02/index.html)
> * [goim中的 bilibili/discovery (eureka)基本概念及应用](/tech/goim-go-03/index.html)
> * [goim 的 data flow 数据流](/tech/goim-go-04/index.html)
> * [goim的业务集成(分享会小结与QA)](/tech/goim-go-06/index.html)
>
>  有个 slack 频道, 不少朋友在交流 goim , 欢迎加入[slack #goim](https://join.slack.com/t/reading-go/shared_invite/enQtMjgwNTU5MTE5NjgxLTA5NDQwYzE4NGNhNDI3N2E0ZmYwOGM2MWNjMDUyNjczY2I0OThiNzA5ZTk0MTc1MGYyYzk0NTA0MjM4OTZhYWE)






## 1. goim的系统集成与拓展

下面画出 goim 定制扩展, 或优化的一个可行方式

 ![goim-architecture-004](/tech/assets/goim-architecture-004.png)

1. 新增 AAA/LB 与 user management sub-system (UMS) , 支持用户注册/激活/权限等相关管理
   , 尤其是 AAA 需要支持用户登录认证成功后, 向用户返回以下数据
   1. goim 需要的 json 格式 token, 指明当前用户可以进入哪个 room , 可以接收哪个 room 的下发消息
   2. 返回 goim 的 comet 地址( 进入 room 接收 im 消息) , 以及 logic 地址 ( 发送 im 消息)
2. 扩展 session server 会话管理, 增加用户上下线状态, 以支持离线消息
3. 在 logic 上或 comet 上扩展, 增加消息存储及相关管理, 保留服务端群发消息( 广播或组播), 支持聊天机器人, 增加即时消息存储或处理接口, 比如离线消息存储, 用户上线后获取离线消息(后台触发发送)
4. 增加 room 管理, 开 im room 聊天室, 切换聊天室, 聊天室内的人员管理/群主(管理员等)
5. 在 comet 上定制扩展,  client 端增加消息发送, 可双向流式发送/接收即时消息



## 2. 认证集成

### 2.1 与 goim 交互前(登录验证及LB分流)

goim 客户端, 或叫 goim 终端, 在与 goim 交互前, 应该与 AAA 或类似网元进行交互:

1. 通过用户/密码或相关信息进行登录, 获取当前用户 ID 及相关认证信息,  参见下一小节的 token 
2. AAA 返回给 goim 终端必要的路由数据, 比如当前 goim 应该去访问哪一个 comet ( 接收 im 消息) , 以及哪个 logic ( 发送 im 消息) ,  或者给 goim 终端一个 discovery 地址, 让 goim 终端去 discovery 得到 comet / logic 地址 -------------> 这一步算是 load balance 负载均衡吧




#### 2.2 认证token

以下代码取自 /examples/goim-web/public/client.js

```
function auth() {
     var token = '{"mid":123, "room_id":"live://1000", "platform":"web", "accepts":[1000,1001,1002]}'
     
     # 参见 /api/comet/grpc/prototol.go 
     # 参见 func (p *Proto) WriteTo(b *xbytes.Writer)
     
     var headerBuf = new ArrayBuffer(rawHeaderLen);
     var headerView = new DataView(headerBuf, 0);
     var bodyBuf = textEncoder.encode(token);
     
     headerView.setInt32(packetOffset, rawHeaderLen + bodyBuf.byteLength);
     headerView.setInt16(headerOffset, rawHeaderLen);
     headerView.setInt16(verOffset, 1);
     headerView.setInt32(opOffset, 7);
     headerView.setInt32(seqOffset, 1);
     
     ws.send(mergeArrayBuffer(headerBuf, bodyBuf));

     ....
   }
```

-----

可见 goim 进行 websocket 连接时, 上报了 token, 如下

```
{
  "mid": 123,                 ## memberID, 即是当前 goim 会员ID
  "room_id": "live://1000",   ## goim 会员进入的房间号, 聊天室号, 群号
  "platform": "web",          ## goim 终端类型
  "accepts": [                ## goim 终端用户可接受哪些 room 的 im 消息
    1000,                     ## goim 用户切换 room, 也就在这里处理啦
    1001,
    1002
  ]
}
```

#### 2.3 认证的集成

认证的集成处理有两处

1. 用户在 goim 连接 comet 前, 应获得用户信息, 以便构造 goim 的认证 token , 就是上章节提到的 mid / room_id / accepts 这些数据
2. 在 logic 中处理认证, 在 /internal/logic/conn.go 中的 func (l *Logic) Connect(c context.Context, server, cookie string, token []byte) 函数体内, 处理连接认证的验证, 比如检查 mid 是否存在, 相应的  room_id / accepts 是否有权限进入等, 如果验证通过, 则生成会话数据, 存入 redis 中;  否则返回认证失败 ( 在 comet 中认证失败的处理是关闭 goim 与 comet 的长连接 )




## 3.  im消息发送与处理

im 消息处理, 包括以下

1. 消息的存储
2. 消息的交互处理, 例如聊天机器人, 敏感消息过滤( 社*主义特色)
3. 离线消息的处理
4. 消息发送的 QOS 
5. goim 客户端通过 comet 上行发送


## 4.  QA



1. 什么是 comet 

> comet 【计】：基于 HTTP长连接的“服务器推”技术，是一种新的 Web 应用架构
>
> 基于这种架构开发的应用中，服务器端会主动以异步的方式向客户端程序推送数据，而不需要客户端显式的发出请求。Comet 架构非常适合事件驱动的 Web 应用，以及对交互性和实时性要求很强的应用，如股票交易行情分析、聊天室和 Web 版在线游戏等。
>
> 服务器推送技术(Server Push)是最近Web技术中最热门的一个流行术语，它的别名叫Comet(彗星)。它是继AJAX之后又一个倍受追捧的Web技术。服务器推送技术与最近的流行与AJAX有着密切的关系。         ------------[百度百科](https://baike.baidu.com/item/comet/10206588)
>


> 我个人也挺喜欢用 comet 这个名词, B格不错, 哈哈, 在我的电信相关解决方案, 这词用来取代 adapter / gateway 这两个比较常见的词 , 来代表后端服务与终端的交互连接点,  一般是部署在 proxy 或 load balance 后面, 面向终端/客户端提供接口服务的"后端的前端", 呵呵

2. 关于 kafka / rabbitMQ / activeMQ / palsar / nsq / nats 等MQ的选择

> 在 goim, 默认的 MQ 采用 kafka, 这是 java实现, 有众多成功的大规模大部署长期运作商用案例的一款重量级 MQ 消息队列
>
> 新起之秀是 palsar (java 实现) , 看技术白皮书, 非常不错, 是比肩 kafka 的重量级作品, 但我没用过
>
> activeMQ 评估比较过, 在我的评估报告中,是不推荐
>
> rabbitMQ 用过, MQ 消息中转效率不是特别高, 也比较稳定
>
> nsq c++ 实现, 比较低层, 效率高但很多 mq 相关业务代码得自己写, nsq 作者还用纯c 重写了 nsq , 叫 nano 
>
> nats 是 go 实现, 轻量级, 效率极好, 缺少消息持久化, 是我个人挺喜欢的一款, 部署方便
> 
> 选择 MQ , 我认为除了业务功能之外, 主要还是看运维支持情况, 尤其是在超大规模部署情况下的运维与调优, MQ 与业务的集成,反而是比重很轻的考虑因素. 所以, 我的建议是, 选择自己运维团队/技术团队最熟悉的 MQ , 选择成功商用案例最多, 生态活跃的 MQ .
>
> 在 goim , 正如 B 站选择 kafka 一样, 应该是比较平衡的.
>
> 至于自己业务中使用哪个 MQ , 建议就是, 让运维优先选择吧, 在运维支持下, 能稳定长期运行, 这点更为重要.

3. 关于 goim 中的 gRPC 是不是可以换掉

> 是的, 可以换掉.
>
> 但不太建议.
>
> RPC 或其他通讯中间件, 更换掉 gRPC 很容易, 建议开发团队综合考虑, 使用自己最熟悉最擅长的 rpc 中间件. 如果没有自己最擅长的中间件, 那就保留 gRPC 吧.


4. 是否可以用 goim 作为通信服务噐或游戏中的通讯组件

> 是的, 理论上是可以的
>
> 但不太推荐这么玩
>
> 技术选型是很复杂, 但也很简单的事儿:  那就是充分考虑当前/将来的业务场景(包括业务运行的各个环境......) , 以及运维/运营/研发支撑这些业务的综合成本
> 
> 虽然 goim 有长连接, 也有不错的部署架构, 但毕竟是一个消息发送/弹幕业务场景积累下来的技术模型, 而不是在游戏开发, 或通信中间件的业务/技术积累下的最优选择






_


_


_


_


_


## 5. 结语

goim 系列小文章, 这一篇是终结了.

得益于喜欢开源的朋友们, 一个多月来 360度花式询问与交流, 在闲着的时间, 写了这些小文章.

2019/05/30晚上, 与70多位朋友以视频会议方式就 goim 及周边技术聊了一个半小时, 也是很有趣的经历.

-------

**再一次, 感谢 https://www.bilibili.com 的开源 &  [毛剑](https://github.com/Terry-Mao/) 大神, 及众多开源社区的前辈们,朋友们**

------


## 关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  



_

_

 [tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://zhuanlan.zhihu.com/tsingsonqin), 2019/05/30


