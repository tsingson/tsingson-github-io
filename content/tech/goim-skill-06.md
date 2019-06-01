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





_


_


_


持续补充中............

 

## 关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  



_

_

 [tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://zhuanlan.zhihu.com/tsingsonqin), 2019/05/30


