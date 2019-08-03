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
> *  fasthttp 概述与 Hello World(本文)
> *  fasthttp 客户端与服务端的封装, 日志与路由
> *  fasthttp 所谓 RESTful (兼介绍fastjson)
> *  fasthttp 中间件( 简单认证/ session会话...)
> *  fasthttp 处理 JWT (及 JWT安全性)
> *  fasthttp 对接非标准 web client (作为AAA, 数据加解密)
> *  fasthttp 缓存/proxy代理/反向代理
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
> in a production serving up to 200K rps from more than 1.5M concurrent keep-alive
> connections per physical server.


事实上, 这有点小夸张, 但在一定场景下经过优化部署, 确是有很高的性能. 

近3年来, fasthttp 被我用在几个重大项目(对我而言, 项目有多重大, 与收钱的多少成正比) 中, 这里, 就写一个小系列, 介绍 fasthttp 的实际使用与经验得失.

>  --------------------------------------
>
>  想直接看代码的朋友, 请访问 [我写的 fasthttp-example](https://github.com/tsingson/fasthttp-example)
>
>  ------------------------------------------

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

> 想要使用 fasthttp 的朋友, 请尽量对 http 1.x 协议要很熟悉, 很熟悉.

#### 1.1 HTTP 1.x 协议简述

简单来说, HTTP 1.x 协议, 是一个被动式短连接的 client (请求 request )  - server ( 响应 response) 交互的规范:

1. 协议一般来说, 以 TCP 通讯协议为基础 ( 不谈 QUIC 这个以 udp 为底层实现的变异)
   
    > web client 通过 DNS 把域名转换成 IP 地址后, 与 web server 握手连接, 连接成功后, web client 客户端向 web server 服务端发出请求, 服务端收到请求后, 向 client 客户端应答 
2. 通过 URL / URI 进行导址, 同时, URL/URI 中包含部分数据
    > URL 形式如 http://192.168.1.1:8088/rpc/schedule 
    > 其中 http://192.168.1.1:8080  这部分是通讯协议, 服务器 IP 地址与端口号, 这是前面 TCP 通讯的依据
    > 1.  web 服务器端在 http://192.168.1.1:8080 这个地址上监听, 随时准备接收 web client 请求并应答
    > 1.  web 客户端通过 http://192.168.1.1:8080 这个地址所指定的 web 服务器进行 tcp 连接, 连接成功后, web 客户端向服务器发出 请求数据, web 服务端应答 响应数据
    > 1.  特别注意, 请求数据, 与响应数据, 遵从 HTTP 协议规定的统一格式
3. 在 HTTP 1.x 协议中规定的传输( 请求/应答) 数据格式, 一般称为 HyperText, 是一种文本数据格式,  当然了, 在 TCP 传输时还是二进制数据块 ( 这是使用 fasthttp 的关键点) . 具体数据格式见 1.2 小节
4.  HTTP 协议规定了一些信令, 如下描述, 来区分不同的交互操作
    > 根据HTTP标准，HTTP请求可以使用多种请求方法:
    >
    > * HTTP1.0定义了三种请求方法： GET, POST 和 HEAD方法。 
    > * HTTP1.1新增了五种请求方法：OPTIONS, PUT, DELETE, TRACE 和 CONNECT 方法。
5. 由于 HTTP 协议相关的 MIME 规范, HTTP 1.x 也可以传输图像/音乐/视频等其他数据格式,但这些被传输的真正有效数据都被封装在 http payload 这一部分里, http header 还保留( 只是字段多少, 以及字段中的值不同)  ---------**这是另一个与 fasthttp 关联的另一个要点**

### 1.2 HTTP 1.x 中的请求/响应共同遵从的数据格式

下面看一个 POST 请求


![](https://user-gold-cdn.xitu.io/2019/8/3/16c53a4bbef1495a?w=1512&h=1300&f=png&s=229637)


![](https://user-gold-cdn.xitu.io/2019/8/3/16c53a4eddef1a43?w=2290&h=1170&f=png&s=215294)


请求数据如下, 响应是一样的格式.
在下面的数据中:

    1. 在下面的数据格式中, 特别注意, 中间有一个空行
    1. 空行上半部分, 叫 http header , 下半部分, 叫 http payload 或叫 http body
    1. 在上半部分的 http header 中, 请注意第1,2行
    1. 请对比一下, 下方同时列出的 GET 请求数据

-----
POST 请求数据示例

```
POST /rpc/schedule HTTP/1.1
Host: 192.168.1.1:3001
Content-Type: application/json
Accept: application/vnd.pgrst.object+json
User-Agent: PostmanRuntime/7.15.2
Host: 192.168.1.1:3001
Accept-Encoding: gzip, deflate
Content-Length: 208
Connection: keep-alive

{
  "actual_start_date": "2019-07-29",
  "actual_end_date": "2019-07-29",
  "plan_start_date": "2019-07-29",
  "plan_end_date": "2019-02-12",
  "title": "00002",
  "user_id": 2098735545843717147
}
```
------
GET 请求示例

```
GET /schedule?user_id=eq.2098735545843717147 HTTP/1.1
Host: 192.168.1.1:3001
Content-Type: application/json
User-Agent: PostmanRuntime/7.15.2
Accept: */*
Host: 192.168.1.1:3001
Accept-Encoding: gzip, deflate
Content-Length: 208
Connection: keep-alive

{
  "actual_start_date": "2019-07-29",
  "actual_end_date": "2019-07-29",
  "plan_start_date": "2019-07-29",
  "plan_end_date": "2019-02-12",
  "title": "00002",
  "user_id": 2098735545843717147
}
```


### 1.3 http 1.x 协议小结与开发关联点

这里几句很重要, 所以, 

#### HTTP 1.x 几个基础点:

1. HTTP 1.x 通过 tcp 进行通讯
1. 请求与响应的格式, 数据数据的格式是一样的
    > 特别注意请求数据中的第一行,第二行
    > 特别注意 HTTP header 与 HTTP payload 的那空行分隔
1. 注意 URL/URI 中也包含有数据, 换个话说,在 http://192.168.1.1:3001/schedule?user_id=eq.2098735545843717147 中, 其他部分 **/schedule?user_id=eq.2098735545843717147** 看做请求数据的一部分

从 HTTP 1.x 协议, 可以总结 web 开发的要点

1. 处理 tcp 通讯,  包括: 
	> *  通过 dns 转化域名得到 IP 地址, 包括 ip4 / ip6  地址
	> *  对 tcp 进行通讯重写或优化, 长连接或短连接, 都在这里了
	> *  或对 tcp 进行转发 ( 这是 proxy ) 或劫持, 在 tcp 通讯最底层进行一些特殊操作
1. 对 URL /URI 进行处理, 这是路由寻址
	> * 按 URI 及相关数据特征进行拦截处理, 这是反向代理与缓存
	> * 进行一些 URI 转换, 例如 302 的重定向
	> * 在 URI 中携带小部分数据的组装与处理
1. HTTP 数据处理
	
	> * 对 HTTP header / HTTP payload 进行处理, 这是变化最多的部分, 按业务/功能的不同, 即简单也复杂




#### fasthttp 的性能优化思路
1. 重写了在 tcp 之上进行 HTTP 握手/连接/通讯的 goroutine pool 实现
2. 对 http 数据基本按传输时的二进制进行延迟处理, 交由开发者按需决定
3. 对二进制的数据进行了缓存池处理, 需要开发者手工处理以达到零内存分配


>  --------------------------------------
>
> 好, HTTP 1.x 就简述到这了, 后面**会大量引用到这一章节说的内容**.
>
>  --------------------------------------
>



## 2. ~~fasthttp 非"标准"的争议, 及为什么选择fasthttp~~
这一章节暂时不写了, 需要一点时间进行调整

## 3. 开发环境及创建 project

个人主力用 MacOS 开发, 以下就以 MacOS 为例

### 3.1. go 安装, 环境变量及goproxy 配置

**下载 golang 编译器并安装**

下载地址为 [https://dl.google.com/go/go1.12.7.darwin-amd64.pkg](https://dl.google.com/go/go1.12.7.darwin-amd64.pkg)

任意下载到一个路径下后, 双击安装

或者, 打开一个 Terminal 命令行终端

```
cd ~
mkdir -p ~/go/src/github.com/tsingson/fasthttp-example/hello-world
cd ~/go/src/github.com/fasthttp-example

wget https://dl.google.com/go/go1.12.7.darwin-amd64.pkg

open ./go1.12.7.darwin-amd64.pkg


```

出现 go 的安装界面后, 一路确认就安装完成了


**配置环境变量**

由于我的 MacOS 已经使用 zshell , 所以, 默认全局环境变量在 ~/.zshrc 

打开 ~/.zshrc  加入以下文本

```

export GOBIN=/Users/qinshen/go/bin  #/Users/qinshen 这是我的个人帐号根路径
export GOSUMDB=off
export GOPATH="/Users/qinshen/go"
export GOCACHE="/Users/qinshen/go/pkg/cache"
export GO111MODULE=on
export CGO_ENABLED=1
# export GOPROXY=http://127.0.0.1:3000  # 这一行是本机在 docker 中运行 athens 这个 goproxy
# export GOPROXY=https://athens.azurefd.net  #远程 ahtens goproxy
# export GOPROXY=direct  # 如果能直接访问 golang.org, 那就用这个配置
export GOPROXY=https://goproxy.cn    #中国大陆, 用这个吧

# export GOPROXY=https://proxy.golang.org  # go1.13 推荐的 goproxy, 试手用
export PATH=$PATH:$GOROOT:$GOBIN

export PS1='%d   '

```

**验证go安装**

```
cd ~/go/src/github.com/fasthttp-example
touch ./hello-world/main.go

```

用一个文本编辑器如 sublime text 3 ,  对 ~/go/src/github.com/fasthttp-example/hello-world/main.go 写入以下 go 代码

```
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello World, 中国...")
}

```

**运行验证**

```
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   go run ./hello-world 
Hello World, 中国...
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   

```

done. 完成.




### 3.2 创建基于 go module 的project 

在项目路径在  go/src/github/tsingson/fasthttp-example 下, 直接运行    或 go mod init 

如果项目路径在任意路径下, 例如在 ~/go/fasthttp-example 下, 则运行  go mod init github.com/tsingson/fasthttp-example


以下是运行结果, 及项目结构

```
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   go mod init github.com/tsingson/fasthttp-example
go: creating new go.mod: module github.com/tsingson/fasthttp-example
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   tree .     
.
├── README.md
├── cmd
│   ├── test-client
│   │   └── main.go
│   └── test-server
│       └── main.go
├── go.mod
├── hello-world
│   └── main.go
├── webclient
│   └── client.go
└── webserver
    ├── config.go
    ├── const.go
    ├── handler.go
    ├── middleware.go
    ├── router.go
    ├── server.go
    └── testHandler.go

6 directories, 13 files
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   
```

### 3.3 Hello World 单机版

好了, 把 hello-world/main.go 改成以下代码

```
package main

import (
	"fmt"
	"os"
)

func main() {
	var who = "中国"
	if len(os.Args[1]) > 0 {
		who = os.Args[1]
	}
	fmt.Println("Hello World, ", who)
}

```

运行一下

```
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   go run ./hello-world/main.go tsingson
Hello World,  tsingson
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   go run ./hello-world/main.go 三明智  
Hello World,  三明智
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   
```

很好, 我们得到了一个单机版, 命令行方式的 hello world.  

下面, 我们把 hello world 改成 fasthttp web 版本...............



## 4. 选用 usber-go/zap 日志并简单封装

**导入 uber 的 zap 日志库**

```
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   go get go.uber.org/zap
```

**创建封装的日志库**

```
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   touch ./logger/zap.go
```

写入以下代码

```
package logger

import (
	"os"

	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

// NewConsoleDebug  new zap logger for console
func NewConsoleDebug() zapcore.Core {
	// First, define our level-handling logic.
	highPriority := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool {
		return lvl >= zapcore.ErrorLevel
	})
	lowPriority := zap.LevelEnablerFunc(func(lvl zapcore.Level) bool {
		return lvl < zapcore.ErrorLevel
	})

	// High-priority output should also go to standard error, and low-priority
	// output should also go to standard out.
	consoleDebugging := zapcore.Lock(os.Stdout)
	consoleErrors := zapcore.Lock(os.Stderr)

	// Optimize the console output for human operators.
	consoleEncoder := zapcore.NewConsoleEncoder(zap.NewDevelopmentEncoderConfig())
	// Join the outputs, encoders, and level-handling functions into
	// zapcore.Cores, then tee the four cores together.

	var stderr = zapcore.NewCore(consoleEncoder, consoleErrors, highPriority)
	var stdout = zapcore.NewCore(consoleEncoder, consoleDebugging, lowPriority)

	return zapcore.NewTee(stderr, stdout)
}

// ConsoleWithStack  console log for debug
func ConsoleWithStack() *zap.Logger {
	core := NewConsoleDebug()
	// From a zapcore.Core, it's easy to construct a Logger.
	return zap.New(core).WithOptions(zap.AddCaller())
}

// Console  console log for debug
func Console() *zap.Logger {
	core := NewConsoleDebug()
	// From a zapcore.Core, it's easy to construct a Logger.
	return zap.New(core)
}

```





## 5. 写一个fasthttp 版本的 Hello World
### 5.1 项目结构
```
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   tree .
.
├── README.md
├── cmd
│   ├── test-client
│   │   └── main.go
│   └── test-server
│       └── main.go
├── go.mod
├── go.sum
├── hello-web
│   ├── hello-client
│   │   └── main.go
│   └── hello-server
│       └── main.go
├── hello-world
│   └── main.go
├── logger
│   └── zap.go
├── webclient
│   └── client.go
└── webserver
    ├── config.go
    ├── const.go
    ├── handler.go
    ├── middleware.go
    ├── router.go
    ├── server.go
    └── testHandler.go

10 directories, 17 files
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example  
```
### 5.2 fasthttp Hello World 服务端
直接上代码
```
package main

import (
	"bytes"
	"strconv"

	"github.com/savsgio/gotils"
	"github.com/valyala/fasthttp"
	"go.uber.org/zap"

	"github.com/tsingson/fasthttp-example/logger"
)

func main() {

	var log *zap.Logger = logger.Console()
	var address = "127.0.0.1:3001"

	// -------------------------------------------------------
	//  fasthttp 的 handler 处理函数
	// -------------------------------------------------------
	var requestHandler = func(ctx *fasthttp.RequestCtx) {

		// -------------------------------------------------------
		// 处理 web client 的请求数据
		// -------------------------------------------------------
		// 取出 web client 请求进行 TCP 连接的连接 ID
		var connID = strconv.FormatUint(ctx.ConnID(), 10)
		// 取出 web client 请求 HTTP header 中的事务ID
		 var tid = string( ctx.Request.Header.PeekBytes([]byte("TransactionID")))
		 if len(tid) == 0 {
		 	tid = "12345678"
		 }

		log.Debug("HTTP 访问 TCP 连接 ID  " + connID)

		// 取出 web 访问的 URL/URI
		var uriPath = ctx.Path()
		{
			// 取出 URI
			log.Debug("---------------- HTTP URI -------------")
			log.Debug(" HTTP 请求 URL 原始数据 > ", zap.String("request", ctx.String()))
		}

		// 取出 web client 请求的 URL/URI 中的参数部分
		{
			log.Debug("---------------- HTTP URI 参数 -------------")
			var uri = ctx.URI().QueryString()
			log.Debug("在 URI 中的原始数据 > " + string(uri))
			log.Debug("---------------- HTTP URI 每一个键值对 -------------")
			ctx.URI().QueryArgs().VisitAll(func(key, value []byte) {
				log.Debug(tid, zap.String("key", gotils.B2S(key)), zap.String("value", gotils.B2S(value)))
			})
		}
		// -------------------------------------------------------
		// 注意对比一下, 下面的代码段, 与 web client  中几乎一样
		// -------------------------------------------------------
		{
			// 取出 web client 请求中的 HTTP header
			{
				log.Debug("---------------- HTTP header 每一个键值对-------------")
				ctx.Request.Header.VisitAll(func(key, value []byte) {
					// l.Info("requestHeader", zap.String("key", gotils.B2S(key)), zap.String("value", gotils.B2S(value)))
					log.Debug(tid, zap.String("key", gotils.B2S(key)), zap.String("value", gotils.B2S(value)))
				})

			}
			// 取出 web client 请求中的 HTTP payload
			{
				log.Debug("---------------- HTTP payload -------------")
				log.Debug(tid, zap.String("http payload", gotils.B2S(ctx.Request.Body())))
			}
		}
		switch {
		// 如果访问的 URI 路由是 /uri 开头 , 则进行下面这个响应
		case len(uriPath) > 1:
			{
				log.Debug("---------------- HTTP 响应 -------------")

				// -------------------------------------------------------
				// 处理逻辑开始
				// -------------------------------------------------------

				// payload 是 []byte , 是 web response 返回的 HTTP payload
				var payload = bytes.NewBuffer([]byte("Hello, "))

				// 这是从 web client 取数据
				var who = ctx.QueryArgs().PeekBytes([]byte("who"))

				if len(who) > 0 {
					payload.Write(who)
				} else {
					payload.Write([]byte(" 中国 "))
				}

				// -------------------------------------------------------
				// 处理 HTTP 响应数据
				// -------------------------------------------------------
				// HTTP header 构造
				ctx.Response.Header.SetStatusCode(200)
				ctx.Response.Header.SetConnectionClose() // 关闭本次连接, 这就是短连接 HTTP
				ctx.Response.Header.SetBytesKV([]byte("Content-Type"), []byte("text/plain; charset=utf8"))
				ctx.Response.Header.SetBytesKV([]byte("TransactionID"), []byte(tid))
				// HTTP payload 设置
				// 这里 HTTP payload 是 []byte
				ctx.Response.SetBody(payload.Bytes())
			}

			// 访问路踊不是 /uri 的其他响应
		default:
			{
				log.Debug("---------------- HTTP 响应 -------------")

				// -------------------------------------------------------
				// 处理逻辑开始
				// -------------------------------------------------------

				// payload 是 []byte , 是 web response 返回的 HTTP payload
				var payload = bytes.NewBuffer([]byte("Hello, "))

				// 这是从 web client 取数据
				var who = ctx.QueryArgs().PeekBytes([]byte("who"))

				if len(who) > 0 {
					payload.Write(who)
				} else {
					payload.Write([]byte(" 中国 "))
				}

				// -------------------------------------------------------
				// 处理 HTTP 响应数据
				// -------------------------------------------------------
				// HTTP header 构造
				ctx.Response.Header.SetStatusCode(200)
				ctx.Response.Header.SetConnectionClose() // 关闭本次连接, 这就是短连接 HTTP
				ctx.Response.Header.SetBytesKV([]byte("Content-Type"), []byte("text/plain; charset=utf8"))
				ctx.Response.Header.SetBytesKV([]byte("TransactionID"), []byte(tid))
				// HTTP payload 设置
				// 这里 HTTP payload 是 []byte
				ctx.Response.SetBody(payload.Bytes())
			}
		}

		return

	}
	// -------------------------------------------------------
	// 创建 fasthttp 服务器
	// -------------------------------------------------------
	// Create custom server.
	s := &fasthttp.Server{
		Handler: requestHandler,       // 注意这里
		Name:    "hello-world server", // 服务器名称
	}
	// -------------------------------------------------------
	// 运行服务端程序
	// -------------------------------------------------------
	log.Debug("------------------ fasthttp 服务器尝试启动------ ")

	if err := s.ListenAndServe(address); err != nil {
		log.Fatal("error in ListenAndServe", zap.Error(err))
	}
}

```
### 5.3 fasthttp Hello World 客户端( GET )
看代码
```
package main

import (
	"net/url"
	"os"
	"time"

	"github.com/savsgio/gotils"
	"github.com/valyala/fasthttp"
	"go.uber.org/zap"

	"github.com/tsingson/fasthttp-example/logger"
)

func main() {

	var log *zap.Logger = logger.Console()
	var baseURL = "http://127.0.0.1:3001"

	// 随便指定一个字串做为 web 请求的事务ID , 用来打印多条日志时, 区分是否来自同一个 web 请求事务
	var tid = "12345678"

	// -------------------------------------------------------
	//      构造 web client 请求的 URL
	// -------------------------------------------------------

	var fullURL string
	{
		relativeUrl := "/uri/"
		u, err := url.Parse(relativeUrl)
		if err != nil {
			log.Fatal("error", zap.Error(err))
		}

		queryString := u.Query()

		// 这里构造 URI 中的数据, 每一个键值对
		{
			queryString.Set("id", "1")
			queryString.Set("who", "tsingson")
			queryString.Set("where", "中国深圳")
		}

		u.RawQuery = queryString.Encode()

		base, err := url.Parse(baseURL)
		if err != nil {
			log.Fatal("error", zap.Error(err))
			os.Exit(-1)
		}

		fullURL = base.ResolveReference(u).String()

		log.Debug("---------------- HTTP 请求 URL -------------")

		log.Debug(tid, zap.String("http request URL > ", fullURL))

	}
	// -------------------------------------------------------
	//      fasthttp web client 的初始化, 与清理
	// -------------------------------------------------------
	//  fasthttp 从缓存池中申请 request / response 对象
	var req = fasthttp.AcquireRequest()
	var resp = fasthttp.AcquireResponse()
	// 释放申请的对象到池中
	defer func() {
		fasthttp.ReleaseResponse(resp)
		fasthttp.ReleaseRequest(req)
	}()
	// -------------------------------------------------------
	//      构造 web client 请求数据
	// -------------------------------------------------------
	// 指定 HTTP 请求的 URL
	req.SetRequestURI(fullURL)

	// 指定 HTTP 请求的方法
	req.Header.SetMethod("GET")
	// 设置 HTTP 请求的 HTTP header

	req.Header.SetBytesKV([]byte("Content-Type"), []byte("text/plain; charset=utf8"))
	req.Header.SetBytesKV([]byte("User-Agent"), []byte("fasthttp-example web client"))
	req.Header.SetBytesKV([]byte("Accept"), []byte("text/plain; charset=utf8"))
	req.Header.SetBytesKV([]byte("TransactionID"), []byte(tid))

	// 设置 web client 请求的超时时间
	var timeOut = 3 * time.Second

	// 计时开始
	t1 := time.Now()

	// DO request
	var err = fasthttp.DoTimeout(req, resp, timeOut)

	if err != nil {
		log.Error("post request error", zap.Error(err))
		os.Exit(-1)
	}
	// -------------------------------------------------------
	//      处理返回结果
	// -------------------------------------------------------
	elapsed := time.Since(t1)
	log.Debug("---------------- HTTP 响应消耗时间-------------")

	log.Debug(tid, zap.Duration("elapsed", elapsed))
	log.Debug("---------------- HTTP 响应状态码 -------------")

	log.Debug(tid, zap.Int("http status code", resp.StatusCode()))
	log.Debug("---------------- HTTP 响应 header 与 payload -------------")

	// -------------------------------------------------------
	// 注意对比一下, 下面的代码段, 与 web server  中几乎一样
	// -------------------------------------------------------
	{
		// 取出 web client 请求中的 HTTP header
		{
			log.Debug("---------------- HTTP header 每一个键值对-------------")
			resp.Header.VisitAll(func(key, value []byte) {
				// l.Info("requestHeader", zap.String("key", gotils.B2S(key)), zap.String("value", gotils.B2S(value)))
				log.Debug(tid, zap.String("key", gotils.B2S(key)), zap.String("value", gotils.B2S(value)))
			})

		}
		// 取出 web client 请求中的 HTTP payload
		{
			log.Debug("---------------- HTTP payload -------------")
			log.Debug(tid, zap.String("http payload", gotils.B2S(resp.Body())))
		}
	}

}

```
### 5.4 编译与运行

**编译**
```
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example   go install ./hello-web/...
/Users/qinshen/go/src/github.com/tsingson/fasthttp-example 
```
**运行**

客户端
 ```
 /Users/qinshen/go/bin   ./hello-client 
2019-08-03T22:48:21.939+0800	DEBUG	---------------- HTTP 请求 URL -------------
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"http request URL > ": "http://127.0.0.1:3001/uri/?id=1&where=%E4%B8%AD%E5%9B%BD%E6%B7%B1%E5%9C%B3&who=tsingson"}
2019-08-03T22:48:21.941+0800	DEBUG	---------------- HTTP 响应消耗时间-------------
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"elapsed": "939.037µs"}
2019-08-03T22:48:21.941+0800	DEBUG	---------------- HTTP 响应状态码 -------------
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"http status code": 200}
2019-08-03T22:48:21.941+0800	DEBUG	---------------- HTTP 响应 header 与 payload -------------
2019-08-03T22:48:21.941+0800	DEBUG	---------------- HTTP header 每一个键值对-------------
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Content-Length", "value": "15"}
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Content-Type", "value": "text/plain; charset=utf8"}
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Server", "value": "hello-world server"}
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Date", "value": "Sat, 03 Aug 2019 14:48:21 GMT"}
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Transactionid", "value": "12345678"}
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Connection", "value": "close"}
2019-08-03T22:48:21.941+0800	DEBUG	---------------- HTTP payload -------------
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"http payload": "Hello, tsingson"}
/Users/qinshen/go/bin   
 ```

服务端
```
/Users/qinshen/go/bin   ./hello-server 
2019-08-03T22:48:12.234+0800	DEBUG	------------------ fasthttp 服务器尝试启动------ 
2019-08-03T22:48:21.940+0800	DEBUG	HTTP 访问 TCP 连接 ID  1
2019-08-03T22:48:21.940+0800	DEBUG	---------------- HTTP URI -------------
2019-08-03T22:48:21.940+0800	DEBUG	 HTTP 请求 URL 原始数据 > 	{"request": "#0000000100000001 - 127.0.0.1:3001<->127.0.0.1:51927 - GET http://127.0.0.1:3001/uri/?id=1&where=%E4%B8%AD%E5%9B%BD%E6%B7%B1%E5%9C%B3&who=tsingson"}
2019-08-03T22:48:21.940+0800	DEBUG	---------------- HTTP URI 参数 -------------
2019-08-03T22:48:21.940+0800	DEBUG	在 URI 中的原始数据 > id=1&where=%E4%B8%AD%E5%9B%BD%E6%B7%B1%E5%9C%B3&who=tsingson
2019-08-03T22:48:21.940+0800	DEBUG	---------------- HTTP URI 每一个键值对 -------------
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"key": "id", "value": "1"}
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"key": "where", "value": "中国深圳"}
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"key": "who", "value": "tsingson"}
2019-08-03T22:48:21.940+0800	DEBUG	---------------- HTTP header 每一个键值对-------------
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"key": "Host", "value": "127.0.0.1:3001"}
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"key": "Content-Length", "value": "0"}
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"key": "Content-Type", "value": "text/plain; charset=utf8"}
2019-08-03T22:48:21.940+0800	DEBUG	12345678	{"key": "User-Agent", "value": "fasthttp-example web client"}
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Accept", "value": "text/plain; charset=utf8"}
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"key": "Transactionid", "value": "12345678"}
2019-08-03T22:48:21.941+0800	DEBUG	---------------- HTTP payload -------------
2019-08-03T22:48:21.941+0800	DEBUG	12345678	{"http payload": ""}
2019-08-03T22:48:21.941+0800	DEBUG	---------------- HTTP 响应 -------------
```
 


## 6. 小结

对比第1章节,第 5章节 以 fasthttp 实现了一个 web 版本的 hello world:
1. fasthttp web 客户端处理请求:
    > * 构造了 URL, 并在 URL 中加上 key=value 键值对的数据
    > * 设置 HTTP header, 注意 header 中的 TransactionID 字段
    > * 设置了 GET 请求方法
    > * 多余设置了 HTTP payload ------------ 注意, 服务器把 GET 方法的 HTTP payload 丢掉了
    > * ------ 发出请求
1. fasthttp web 服务端处理响应:
    > * 设置了 HTTP status code 
    > * 设置 HTTP header, 注意 header 中的 TransactionID 字段
    > * 设置 HTTP payload 
    > * ------ 发出响应
1. fasthttp 处理请求与响应, 遵从了 HTTP 规范, 非常相似

那么, 看起了很繁琐, 好吧, 后续文章我们再来谈, 如何简化, 以及如何得到高性能执行效率的同时, 能提高开发效率


_


_


_

## 7. 推荐
* 推荐谢大的小工具 [https://github.com/astaxie/bat](https://github.com/astaxie/bat)
* [](https://www.cnblogs.com/qcrao-2018/p/10285348.html)
_

### 关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  


_

_
 [tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://tsingson.github.io/music/about-studio/),   2019/08/02





