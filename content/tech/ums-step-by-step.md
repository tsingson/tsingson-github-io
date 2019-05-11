---
title: "go-ums 从设计到实现( v0.1.0 )-持续更新"
date: 2019-05-11T12:02:57+08:00
hidden: false
draft: false
author: "tsingson"
categories : [ "Development" ]
series: "goim"
tags: [goim,golang, ums, aaa, register, grpc, flatbuffers, android, websocket, RESTful]
keywords: [tsingson]

description: "go-ums 从设计到实现( v0.1.0 )"
slug: "ums-step-by-step"
---
![self](/tech/assets/self.jpg)

[简述]  一个从零开始的小项目, 持续推进. go-ums 开发目标是一个开源项目, 核心由 golang 开发,  提供用户管理(user-management-subsystem) / AAA 认证/鉴权/授 / 多业务会话共享与管理等, 以支持分布式部署及云部署为主要目标
<!--more-->






> 项目源码 [https://github.com/tsingson/go-ums](https://github.com/tsingson/go-ums)


## 0. go-ums 是什么

go-ums 开发目标是一个开源项目, 核心由 golang 开发,  提供用户管理(user-management-subsystem) / AAA 认证/鉴权/授 / 多业务会话共享与管理等, 以支持分布式部署及云部署为主要目标

这是一个从零开始的小项目, 持续渐进

在 [reading-go 夜读](https://github.com/developer-learning/reading-go) 相关 issue 讨论中, 有个 gin 的练手项目, 有点意思.

这是一个类似的项目, **不同的地方是, 这个项目是以文档开始的.**

> 更多项目关联文档, 请访问 [https://github.com/tsingson/go-ums](https://github.com/tsingson/go-ums) , 在 readme 中列示并持续添加中


**先有文档, 再写代码. 在文档简单说明一些我的设计套路**


> 注:  这里的设计, Design, 不限于架构设计, 程序设计, 也包含个人在摄影/印刷/书籍/海报中找出问题/解决问题的一些"套路". 
>
> 先文档, 后实施, 也是曾经作 SA/SE 的习惯了吧. 当年曾戏称自己是 Simple Editor , 哈, 怀念那8年, 大胡子, yaho BB,  black more, 北京香山, 上海文广,三角洲岛.......
>
> 稍后在持续更新中一一道来, 看看横向之间的有趣关联.

## 1. 业务场景

这就是原始需求的收集/整理/清理/梳理, 是开发贯串始终的目标与仲裁准则:

1. 是什么? 有什么价值?
2. 是谁在用? 如何用( 业务操作流程)? 哪些是关键节点(不能省掉的业务行为), 重点与难点是什么?
3. 在这些业务场景中, 边界在哪里( 哪些情况是正常, 哪些是异常, 如何判定, 什么时间什么情况下有哪些处理方式)



从一个简单而典型的业务场景开始, 来, 动动脑, 再动动脑,  

### 1.1 业务场景/需求描述

这个开源项目是从 goim 在开源社区交流讨论后, 偶然开始的


从业多年, 个人几乎不在网络上谈论个人在 iT 技术职业经历, 尤其是项目细节, 这里, 算是换个方式来公开谈谈吧


所以, 这里也从 goim 作为起点( 也仅是个起点)


很多朋友, 在阅读 goim 源码时,  都会感觉到, 就算是 im 业务完整实施, 需要与用户管理结合:

1. 用户注册后, 提供一个用户唯一标识, , 对应 goim 中的 mid 
2. 用户认证, 让用户可以长连接到 goim , 在认证成功时 
    1. 提供一个 token , 该 token 中包含用户可以进入的房间列表
    2. 提供用户发送 im 信息的入口地址
4. 用户在 goim 中涉及会话的业务功能( 变更房间.....)

所以, 我们可以从上面描述到的, (不能省掉的) 基本业务功能开始



来, 来, 来 ...GO!





#### 1.1.1 场景与流程简述

( 省略)

稍后补充示意图

#### 1.1.2 业务指标(性能要求)

(简化)  
1. 要求业务响应, 排除网络延迟, 业务响应要求在 100ms 以内
4. 支持10K tps 高并发
5. 年度业务中断时间低于5次, 业务中断时间低于30分钟
6. 业务数据保留期限 1年
5. 业务操作记录要求支持审计

#### 1.2.3 部署要求

(简化) 
1. 服务器部署中国骨干网络IDC (电信网络为主, 广州/上海两地IDC ) 及美国骨干网络IDC ( 略)
2. 以部署服务端优先支持 linux 与 docker 
3.  客户端支持 windows / mac / linux 的 terminal 命令行, 支持主流浏览器( 当前市场占80%的主流版本, 相当于 IE 10+)
#### 1.2.4 开发/运维/运营要求

(省略)

#### 1.2.5 成本要求
(省略)

### 1.2  feature list 需求列表
注, 以下按开发进度作了分组. 

分组的意思, 就是排优先级, 排优先级即是排重要/紧急程度(按 **SWOT** Analysis 方式), 以决定先后处理顺序.



#### 1.2.1 prototype 开发需求 v0.1.0

用户相关操作

 - [ ] 用户注册 register 
    用户以 email + 密码 注册提交个人信息, 提交后验证 email 是否存在唯一冲突, 如果没有, 注册成功( 分配用户ID)
注意, 这里隐藏着一个检查用户是否存在的操作
 - [ ] 用户登录与登出 login / logout 
    用户以  email + passwrd 或 用户名 + password 或 userID + password 可以进行登录, 登录成功, 给用户发送通行令牌 accessToken 
    注意, 这里隐藏着一个检查用户登录成功返回一个 access token 的操作
 - [ ] 用户认证 authentication
    针对具体业务, 用户访问具体业务时, 发送 access token 进行认证, 如果用户认证成功, 可以访问指定业务, 否则拒绝. 一般来说, 用户认证成功即表示创建一个合法的业务会话
 - [ ] 用户通行令牌验证 verify
    用户访问指定业务时, 验证 token 是否在有效期内, 以及相关授权是否有变更, 如不符合业务授权限制, 则拒绝提供服务


### # 1.2.2  trial 试运行 v0.1.1 
用户相关操作
 - [ ] 用户激活 active
    用户注册后, 向用户邮箱发送 email 验证的激活邮件, 用户从邮件中获取激活码后, 进行激活操作 
 - [ ] 用户登录与登出 login / logout 
    用户以  email + passwrd 或 用户名 + password 或 userID + password 可以进行登录, 登录成功, 给用户发送通行令牌 accessToken 
  - [ ] 用户锁定 suspend
    禁止用户业务消费行为, 一般来说, 用户在具体业务上, 会有指定的服务周期, 超出服务周期, 用户被锁定
  - [ ] 用户恢复 resume
    恢复用户授权的业务行为
  - [ ] 用户删除 deleted 
    用户删除自己或管理后台删除用户, 注意, 在商用中, 用户“删除”一般意味着用户状态修改为 deleted 并停止所有访问与所有业务, 但保留所有关联数据( 这些关联数据, 只有在一定期限后, 会手动或自动清除 purge )

## 2. 概要设计

从 1.1 小节, 很容易作一个简单设计

### 2.1 数据模型

用户对象名称  account 
> 1. 用户ID, 与 goim 配合, 就用 int64 吧, 一个全局唯一的正整数
> 2. email , (简化), 都是字符串,  email 格式也不验证, 其中 email 字符串长不得少于5个字符( >=5 ) ,
> 3.  password,  (简化)  都是字符串,  密码不得少于6个字符( >= 6), 字符为 a..zA..Z0..9$%-
> 4. access tokdn 通告令牌, 也用字符串代替吧, 字符串长度 >=32 字符
> 5. 用户角色, 分别为 非会员, 会员
> 6. 用户状态, 分别为注册未激活, 已激活, 禁用, 已删除
> 7. 用户创建时间, 简单点, 用 int64 或  UTC timestamp 
> 8. 用户信息变更时间, 简单点, 用 int64 或  UTC timestamp 
>


操作结果(状态)定义
> 1. transaction ID 事务唯一标识
> 2. 状态码, 整数, 操作成功返回 200 / 操作失败返回 500 / 请求接受并在处理中返回202
> 3. 操作结果文本信息, 少于255字符, 操作成功,  文本信息, 要求标记业务操作名称, 如 register ), 操作失败, 返回失败原因( 文本信息 )

注:   transaction ID 事务唯一标识, 支持异步操作, 以支持分布式或集群, 以及操作失败时, 进行操作重试( 这样可以让服务端处理上次操作失败的数据, 例如进行清理, 或继续使用, 以及在日志中进行检查/审计)

原型实现的约束与限制:

> 1. 为了快速实现原型, 用户密码直接存储, 不加密
> 2. 用户ID , 直接用用户创建的 utc 时间截( 纳秒 )
> 3. ~~简化操作结果(状态定义),  这里先按 golang 实现习惯方式, 操作结果修改为 go 的 error 返回( 即 error = nil 或 != nil )  注: 这个地方, 只是针对 go 的简化~~

### 2.2 用户操作
(简化)

> 1. 用户注册  register ------> 调用 exists 检查是否重复注册, 如果不是, 创建用户ID 并保存用户信信息, 返回用户数据( 不返回密码), 返回操作结果状态码
> 2. 检查用户是否重复存在 exists, 返回操作结果
> 3. 用户登录 login ------> 以 email + pwd 登录, 登录成功, 创建并返回 access token, 返回操作结果状态码
> 4. 用户登出 logout -----> 清除 access token , 返回操作结果状态码
> 5. 用户认证 auth -------> 输入用户ID 或 token  , 调用 verify 检查 会话中的token 是否合法, 如果 token 合法则返回成功, 返回操作结果状态码
> 6. 用户令牌验证 verify --------> 检查 token 是否存在, 并且是否相等, 返回操作结果状态码



### 2.3 详细设计 (仅以 register 用户注册为例)

#### 2.3.1 register 用户注册(示例)

##### ----> 输入:
1. transaction ID 事务唯一标识
2. email  合法邮箱地址
2. password 密码

##### ----> 内部逻辑实现
1. 检查 email 是否重复( 即被成功注册过), 判断没有重复, 则进行 2, 判断有重复, 进行下说明 3
2. 生成用户 ID , 并保存到文件或数据库, 保存成功, 返回成功结果输出, 保存失败, 进行下说明 3
3. 处理错误输出
4. 以上处理, 每一步均以 事务 ID 与时间为主键存日志

##### ----> 输出
######  输出 (成功, 返回用户信息, 屏蔽密码字段)

1. account ID 用户ID
2. email
6. 用户角色
7. 用户状态
8. 创建时间
9. 变更时间
##### 输出 (失败, 返回事务标识, 错误状态码, 错误原因文本)

1. transaction ID 事务唯一标识
2. 返回操作结果状态码
3. 操作结果文本描述

#### 2.3.2 接口设计
设计实现以下接口
1. RESTful 接口 ( 描述省略, 见代码)
2. gRPC ( protobuf ) 接口 ( 略 )
3. gRPC ( flatbuffers ) 接口( 对接 android java SDK , 详细说明略)
4. websocket 接口 ( 略 )
5. TCP 接口 ( 略 )

#### 2.3.3 测试用例如下
1. 检查 email 无重复, 操作成功
2. 检查 email 有重复, 操作失败
3. 异常, 主要关注两个, 保存失败或异常,  通讯异常中断 (详细说明, 省略)

## 3.设计实现 ( golang 为例)

### 3.1 模型的实现
用户模型与操作的 golang 实现

 见 [https://github.com/tsingson/go-ums/blob/master/model/account.go]


 (说明省略)
### 3.2 接口设计的实现

 见 [https://github.com/tsingson/go-ums/blob/master/model/account.go]

(说明省略)


### 3.3 业务逻辑实现

见 [https://github.com/tsingson/go-ums/blob/master/pkg/services/account.go](https://github.com/tsingson/go-ums/blob/master/pkg/services/account.go)



(说明省略)
### 3.4 单元测试
测试用例如下
1. 检查 email 无重复, 操作成功
2. 检查 email 有重复, 操作失败
3. 其他异常( 略)

参见  [go-ums v0.1.0 测试/编译/运行](./build-test.md)

### 3.5 集成测试
省略..........

## 4. 性能测试/部署测试/  trial 验证

省略 ...

## 5. 附注/参考

持续...

_

### 关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  


_

_
 [tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://tsingson.github.io/music/about-studio/),   2019/05/11