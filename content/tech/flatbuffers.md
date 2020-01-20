

---
title: "websocket ( go srv / JS client) 使用flatbuffers 交互"
date: 2020-01-02T22:02:57+08:00
hidden: false
draft: false
author: "tsingson"

categories : [ "Development" ]
series: "gRPC"
tags: [gRPC, flatbuffers, websocket, golang]
keywords: [tsingson,gRPC, flatbuffers, websocket, golang]

description: "在 go websocket server 与 javascript websocket client 交互中使用 flatbuffers "
slug: "flatbuffers"

---



![photo-desk](/tech/assets/photo-desk.jpg)

[简述]   在 go websocket server 与 javascript websocket client 交互中使用 flatbuffers 
<!--more-->



 



> 代码在 [https://github.com/tsingson/fastws-example](https://github.com/tsingson/fastws-example)


## 0. 简要说明
为某个开源项目增加 websocket 对接, 写了这个示例

代码中 javascript 对 flatbuffers 的序列化/反序列化, 查了一天资料, 嗯哼, 最终完成了.
看代码吧.........

-----------
### 0.1 关于序列化/反序列化
序列化 serialized / 反序列化 un-serialized , 一般是指对像转成二进制序列( 或叫二进制数组), 以及逆向从二进制转成对像的过程, 一般用在几个地方
1. 网元之间传输.  比如 RESTfull 是在 HTTP 协议( HTML over TCP ) 上进行交互时使用 JSON 数据格式进行序列化与反序列化; 比如 gRPC 默认采用 protobuffers 在 HTTP2 传输上进行数据序列化与反序列化;
2. 对象数据持久化存储到文件系统, 以及从文件系统读取到对象时;
3. 异构开发SDK或API之间交互或共享数据, 比如 go语言调用底层 c++ 库........

### 0.2 关于 flatbuffers 
flatbuffers 是 google 员工在开发游戏过程中, 仿 protobuffers 写的一个高性能序列化/反序列化库, 通过 IDL (接口描述语言) 定义, 通过 flatc 编译为多种语言的接口对象序列化/反序列化的强类型库, 支持 c++ / c / java / python / rust / js / typescript / go / swift.........

fastbuffers 的介绍, 参见 [于德志](https://halfrost.com/) 的文章 [https://halfrost.com/flatbuffers_schema/](https://halfrost.com/flatbuffers_schema/), 几篇文章写得很细致,精确,完整
> **PS: [于德志](https://halfrost.com/) 的技术专题介绍文章,语言简练易懂, 配图简单明了,非常值得一读**
> 
> 
> 说起来, 我的英文阅读能力还可以, 但不得不说, 访问  [于德志](https://halfrost.com/) 的 [https://halfrost.com/tag/protocol/ 协议相关专题文章](https://halfrost.com/tag/protocol/) 还是很愉悦轻松. 谢谢了!



 flatbuffers 的特点, 个人见解:

1.  flatbuffers 的序列化, 慢于 protobuffers ( 约是 protobuffers 的两倍耗时) , 与 JSON 相仿, 甚至有时慢于 json 
2.  flatbuffers 的反序列化, 约10倍快于 protobuffers, 当然也就快于 JSON 了
3.  flatbuffers 在反序列化时, 是内在零拷贝, 序列化后的数据与内存中是一致的, 这让 flatbuffers 序列化后的二进制数据直接导入内存, 以及从内存中读取时都非常快

所以, 在一次序列化, 而多次反序列化的情况下, 以及对反序列化要求速度非常快的情况, 可以考虑选择 flatbuffers , 想想 google 员工为游戏而开发 flatbuffers 这一典型场景吧

### 0.3 我在哪里使用( 或计划使用 ) flatbuffers ? 
在以下场景中, 我使用了( 或正在计划使用) flatbuffers:

1. Sub/Pub 订阅/发布的消息系统. 在某些 Sub/Pub 场景中, Pub 时序列化消息对象, 尤其是 flatbuffers 中的 union , 挺好用.  ------------- 而在 Sub 订阅消费端, 尤其多端消费, 高效的反序列化, 可以减少最多达1/4, 平均1/5 左右时延 (注: 仅是个人应用场景的经验值, 供参考)
2. 内存缓存( 包括 session 会话数据) , 某些应用中的内存缓存需要持久化, 这些内存缓存通过并发保存到多个文件后, 在应用重启时从文件中重建缓存, 非常快
3. IM 即时通讯, 以及某些情况下的 gRPC, 这个与第一条类似. 参见我以前的文章 [GOIM的架构与定制](https://juejin.im/post/5cbb9e68e51d456e51614aab)------ 事实上, 这一篇文章, 正是为定制开发的 IM 而准备. --------- 至于 gRPC , 是的, gRPC 默认的 ptotobuffers 可以用 flatbuffers 更换, 我在几个商用项目中使用, 某商用项目中的 gRPC + flatbuffers 已经上线运行一年了.

### 0.4 flatbuffers 的重大改进

之前, flatbuffers 在序列化时代码很让人着急, 但2019年12月的一个改进, 让 flatbuffers 序列化时代码简化不少

```
flatc --gen-object-api ./*.fbs 
```

以上参数的添加, 让 flatbuffers 序列化简单如下:

```
// --------------- 这是 fbs 文件中的 IDL 
  table LoginRequest{
  msgID:int=1;
  username:string;
  password:string;
  }
  
// -------------- 这是 flatc 编译后的 go 代码
type LoginRequestT struct {
	MsgID    int32
	Username string
	Password string
}

func LoginRequestPack(builder *flatbuffers.Builder, t *LoginRequestT) flatbuffers.UOffsetT {
	if t == nil {
		return 0
	}
	usernameOffset := builder.CreateString(t.Username)
	passwordOffset := builder.CreateString(t.Password)
	LoginRequestStart(builder)
	LoginRequestAddMsgID(builder, t.MsgID)
	LoginRequestAddUsername(builder, usernameOffset)
	LoginRequestAddPassword(builder, passwordOffset)
	return LoginRequestEnd(builder)
}

// ----------- 这是我做的简单封装
func (a *LoginRequestT) Byte() []byte {
	b := flatbuffers.NewBuilder(0)
	b.Finish(LoginRequestPack(b, a))
	return b.FinishedBytes()
}


//---------------- 这里是序列化
	l := &LoginRequestT{
		MsgID:    1,
		Username: "1",
		Password: "1",
	}

	b := l.Byte()  // ------------- 变量 b 是序列化后的二进制数组
	
	
```




## 1. 使用代码库

示例代码使用了以下开源库
* [fasthttp](http://github.com/valyala/fasthttp) 
* [fasthttp router](https://github.com/fasthttp/router) 
*  [fastws ](https://github.com/fasthttp/fastws) ----  fasthttp 实现的 websocket 库
*   [flatbuffers](https://github.com/google/flatbuffers) ---- flatbuffers 高效反序列化通用库, 用在 go语言/javascript 
*   [websockets/ws](https://github.com/websockets/ws) ---- javascript websocket 通用库

## 1. flatbuffers  IDL 示例 
xone.fbs 示例来自 [https://www.cnblogs.com/sevenstar/p/FlatBuffer.html](https://www.cnblogs.com/sevenstar/p/FlatBuffer.html), 感谢!!

```
namespace xone.genflat;

  table LoginRequest{
  msgID:int=1;
  username:string;
  password:string;
  }

  table LoginResponse{
 msgID:int=2;
 uid:string;
 }

 //root_type非必须。

 //root_type LoginRequest;
 //root_type LoginRespons
```

## 2. flatc 编译代码

生成 javascript

```
flatc -s --gen-mutable ./*.fbs
```



生成 golang

```
flatc  --go --gen-object-api --gen-all  --gen-compare  --raw-binary ./*.fbs
```

## 3. 主要代码说明

```
./cmd/wsserver/main.go ----- websocket server 
./cmd/wsclient/main.go ----- websocket client
./ws/... -------------------  websocket go code for websocket handler and websocket client 
./jsclient/ws.js  ---------- javascript client code , please check-out package.json for depends
```




## 4. javascript 序列化/反序列化

**请注意代码注释中的--------- 特别注意这一行**

```
// ------------ ./jsclient/index.js

const flatbuffers = require('./flatbuffers').flatbuffers;
const xone = require('./xone_generated').xone; //Generated by `flatc`.

//-------------------------------------------
//  serialized
//-------------------------------------------
let b = new flatbuffers.Builder(1);
let username = b.createString("zlssssssssssssh");
let password = b.createString("xxxxxxxxxxxxxxxxxxx");
xone.genflat.LoginRequest.startLoginRequest(b);
xone.genflat.LoginRequest.addUsername(b, username);
xone.genflat.LoginRequest.addPassword(b, password);
xone.genflat.LoginRequest.addMsgID(b, 5);
let req = xone.genflat.LoginRequest.endLoginRequest(b);
b.finish(req); //创建结束时记得调用这个finish方法。


let uint8Array = b.asUint8Array();   // ------------- 特别注意这一行

console.log(uint8Array);
// console.log(b.dataBuffer() );
//-------------------------------------------
//  un-serialized
//-------------------------------------------
let bb = new flatbuffers.ByteBuffer(uint8Array);  //-------------- 特别注意这一行
let lgg = xone.genflat.LoginRequest.getRootAsLoginRequest(bb);


console.log("username: ", lgg.username());
console.log("password", lgg.password());
console.log("msgID: ", lgg.msgID());

```



## 5.  golang 中对 flatbuffers 的序列化/反序列化

```

// ------ ./apis/genflat/model.go

func (a *LoginRequestT) Byte() []byte {
	b := flatbuffers.NewBuilder(0)
	b.Finish(LoginRequestPack(b, a))
	return b.FinishedBytes()
}

func ByteLoginRequestT(b []byte) *LoginRequestT {
	return GetRootAsLoginRequest(b, 0).UnPack()
}


// ------- ./apis/genflat/model_test.go

func TestLoginRequestT_Byte(t *testing.T) {
	as := assert.New(t)
	// serialized
	l := &LoginRequestT{
		MsgID:    1,
		Username: "1",
		Password: "1",
	}

	b := l.Byte()

	// un-serialized 
	c := ByteLoginRequestT(b)
	if l.MsgID > 0 {
		fmt.Println(" id > ", c.MsgID, " u > ", c.Username, " pw > ", c.Password)
	}

	as.Equal(l.Password, c.Password)

}

```

## 6. websocket 代码
```

ws.onmessage = (event) => {
    //-------------------------------------------------------------------
    //   read from websocket and un-serialized via flatbuffers
    //--------------------------------------------------------------------
    let aa = str2ab(event.data);
    let bb = new flatbuffers.ByteBuffer(aa);
    let lgg = xone.genflat.LoginRequest.getRootAsLoginRequest(bb);
    let pw = lgg.password();

    if (typeof pw === 'string') {
        console.log("----------------------------------------------");

        console.log("username: ", lgg.username());
        console.log("password", lgg.password());
        console.log("msgID: ", lgg.msgID());
    } else {
        console.log("=================================");
        console.log(event.data);
    }


    // console.log(`Roundtrip time: ${Date.now() }` , ab2str(d ));

    setTimeout(function timeout() {
    //-------------------------------------------------------------------
    //   serialized via flatbuffers and send to websocket 
    //--------------------------------------------------------------------
        let b = new flatbuffers.Builder(1);
        let username = b.createString("zlssssssssssssh");
        let password = b.createString("xxxxxxxxxxxxxxxxxxx");
        xone.genflat.LoginRequest.startLoginRequest(b);
        xone.genflat.LoginRequest.addUsername(b, username);
        xone.genflat.LoginRequest.addPassword(b, password);
        xone.genflat.LoginRequest.addMsgID(b, 5);
        let req = xone.genflat.LoginRequest.endLoginRequest(b);
        b.finish(req); //创建结束时记得调用这个finish方法。


        let uint8Array = b.asUint8Array();

        ws.send(uint8Array);
    }, 500);
};

function str2ab(str) {
    let array = new Uint8Array(str.length);
    for (let i = 0; i < str.length; i++) {
        array[i] = str.charCodeAt(i);
    }
    return array
}

```



## 6. 参考

*  [https://github.com/google/flatbuffers/issues/3781](https://github.com/google/flatbuffers/issues/3781)

 

## 7. 其他

macOS 下从源码编译 flatc 
```
git clone https://github.com/google/flatbuffers

cd github.com/google/flatbuffers

cmake -G "Xcode" -DCMAKE_BUILD_TYPE=Release

cmake --build . --target install
 
```

