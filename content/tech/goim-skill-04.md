---
title: "goim 中的 data flow 数据流转及优化思考"
date: 2019-05-07T12:02:57+08:00
hidden: false
draft: false
author: "tsingson"
categories : [ "Development" ]
series: "goim"
tags: [goim,golang]
keywords: [tsingson]

description: "goim 中的 data flow 数据流转及优化思考"
slug: "goim-go-04"
---

[简述]  [http://goim.io](http://goim.io) 是 非常成功的 IM (Instance Message) 即时消息平台 , 本文介绍 goim 中的数据定义与 data flow 数据流转
<!--more-->


## 1. goim 中的 data flow 数据流转
### 1.1 架构中的数据流转
看图

![goim-architecture-dataflow](/tech/assets/goim-architecture-dataflow.png)

数据流转
1. http 接口向 logic 发送数据
2. logic 从 redis 中获取会话数据, 以 protobuf 序列化数据, 发送到 MQ 
3. job 从 MQ 中订阅数据, 取出 im 发送数据中的 server / room 向指定 server 的 comet 发送数据
4. comet 接收 job 分发的数据后, 存入 指定 channel 的 ring buffer , 再转为 tcp/websocket 数据包, 发送到指定 channel 的客户端



## 1.2 简化后的数据流转细节

![goim-architecture-dataflow-detail](/tech/assets/goim-architecture-dataflow-detail.png)

上示意图标注了 goim 中的关键数据结构:
1. 标注了 im 发送数据构成, 注意, 这个数据结构是被logic 以 protobuf 序列化后发到 MQ , 并在 job 中反序列化后, 分发到 comet 
2. 这里的会话信息, 主要是 mid --> server 与 room-->server 的对应关系, 存在 redis 中
3. comet 中的 im 信息, 由 job 从 MQ 中反序列化后, 取出 server / room / keys( 一到多个key , 对应 channel ) 发送到指定 comet server 
4. comet 以 tcp / websocket 封装数据包, 发送给终端用户, 终端解包后显示



## 2. goim 中的数据定义
### 2.1. logic 发送 im 信息
发布 im 信息定义( 在 protobuf 中的定义)
```
message PushMsg {
    enum Type {
        PUSH = 0;
        ROOM = 1;
        BROADCAST = 2;
    }
    Type type = 1;
    int32 operation = 2;
    int32 speed = 3;
    string server = 4;
    string room = 5;
    repeated string keys = 6;
    bytes msg = 7;
}
```


###  2.2 会话数据

当 tcp client 或 websocket client 连接 comet server 时, comet 以 gRPC 向 logic 进行内部通讯, 生成会话数据, 存在 redis 中, 具体细节不展开, 看代码

当 http client 向 logic 发送 im 消息时, logic 向 redis 查询会话数据, 对于已经存在的 room--> server / mid ( memberID) --> server 即发送消息到 MQ , 该部分代码比较清楚, 也不再加说明






### 2.3.  tcp / websocket 数据包定义

推送 im 信息,  对象名称为 proto,  在 protobuf 中定义
```
message Proto {
    int32 ver = 1 [(gogoproto.jsontag) = "ver"];
    int32 op = 2 [(gogoproto.jsontag) = "op"];
    int32 seq = 3 [(gogoproto.jsontag) = "seq"];
    bytes body = 4 [(gogoproto.jsontag) = "body"];
}
```
> protobuf 文件 [https://github.com/Terry-Mao/goim/blob/master/api/comet/grpc/api.proto](https://github.com/Terry-Mao/goim/blob/master/api/comet/grpc/api.proto) 中第12行


> tcp / websocket 数据包组包/折包操作在 [/api/comet/grpc/protocol.go](https://github.com/Terry-Mao/goim/blob/master/api/comet/grpc/protocol.go)  

![goim-pockage-define](/tech/assets/goim-pockage-define-7216912.png)

由上图可见,  goim 在 tcp /websocket 数据包的数据包定义, 与 go 中 proto 定义, 多了, 数据包总长度 / 包头长度两个字段




## 3. comet 中的处理

![goim-dataflow-detail](/tech/assets/goim-dataflow-detail.png)

简化数据流转, 从发送端数据到 接收端数据, 可以看到,  serverID / roomID / channel ( 用 mid 或 key 来指示) 的主要作用作为分流/分发用, 在最后推送数据包中, 就不在包含这三个字段了.

同时,  comet 中使用了 ring buffer 来缓存一个 channel 送达的多条信息并推送到终端, 这里, 并没有看到对推送下发的信息作更多处理. 



-----

**看代码, 补充细节**

```
// Channel used by message pusher send msg to write goroutine.
type Channel struct {
	c        *conf.CometConfig
	Room     *Room
	CliProto Ring
	signal   chan *grpc.Proto
	Writer   xbufio.Writer
	Reader   xbufio.Reader
	Next     *Channel
	Prev     *Channel

	Mid      int64   // #########   memberID  
	Key      string
	IP       string
	watchOps map[int32]struct{}

	mutex sync.RWMutex
}
```
这里:
1. mid 就是 memberID , 当前 channel ( 用户端与 comet 的长连接) 是哪个用户连接上的
该长连接使用 key 作为长连接的会话标识, 换个方式说, key 也就标定了一个 im 信息要发给哪个/哪几个在线长连接对端的用户
2. key 就是长连接的会话ID, 可以这么理解, 就算是 sessionID 吧
3. watchOps 是一个map 映射表, 其中的 int32 是房间号.  map 多个房间号, map 结构是用来查询房间号是否在 map 中存在或不存在. watchOps 是当前长连接用户用来监听当前客户端接收哪个房间的 im 消息推送, 换个方式说, 一个 goim 终端可以接收多个房间发送来的 im 消息
4. watchOps 初始化是在 tcp / websocket 客户端进行首次连接时处理的, 细节看代码.

--------

从 logic 自 http 的 post 请求中, 获取发布 im 信息后, 序列化发到 MQ, 在 job 中拆包反序列化, 再组包, 这一步骤对性能是否有影响, 需发测试数据来定位, 但个人感觉, 这几次拆包组包, 有点重复.



## 4. 小结
以上, 应开源社区的朋友要求, 对内部数据结构作了一个简化分析, 花时不多,水平有限,  或有考虑不周或分析不当, 欢迎批评指点.

最后,  [http://goim.io](http://goim.io)  在网络上相关文章不少, 好文不少, 给我启迪, 一并感谢.


推荐以下文章:

* [https://alexstocks.github.io/html/im.html](https://alexstocks.github.io/html/im.html) 作者 AlexStcks, 非常棒的文章, 集思践行, 很有深度
* [https://github.com/LinkinStars/simple-chatroom](https://github.com/LinkinStars/simple-chatroom) 作者:  LinkinStars, 这个值得一看
* [https://moonshining.github.io/2018/03/09/goim-comet模块源码分析/](https://moonshining.github.io/2018/03/09/goim-comet模块源码分析/)  写得很细致

-------

**再一次, 感谢 https://www.bilibili.com 的开源 &  [毛剑](https://github.com/Terry-Mao/)  及众多开源社区的前辈们,朋友们**

_

## 关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  


_

_
 [tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://tsingson.github.io/music/about-studio/),   2019/05/07

 