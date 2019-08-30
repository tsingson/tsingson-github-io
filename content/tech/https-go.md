---
title: "用 let's Encrypt 实现 HTTPS 示例( fasthttp 与net/http)"
date: 2019-08-07T12:02:57+08:00
hidden: false
draft: false
author: "tsingson"
categories : [ "Development" ]
series: “golang"
tags: [golang]
keywords: [tsingson]

description: "用 let's Encrypt 实现 HTTPS 示例( fasthttp 与net/http)"
slug: "https-setup"

---



 ![go-logo-music](/tech/assets/go-logo-music.jpg)





[摘要] let's Encrypt 是一个免费提供 HTTPS 的签名服务, 这里提供一个示例,用 [certmagic](https://github.com/mholt/certmagic) 实现 fasthttp 与 net/http 上支持 HTTPS
<!--more-->


## 0. 基本配置

我的域名, 包括 tsingson.io 与 www.tsingson.io , 在 DNS 上加上 A 记录, 指向我的服务器 IP

为了在 let's Encrypt 获取 HTTPS 签名证书, 使用 tsingson_at_me_com 这个邮箱进行管理

服务器上, 把 HTTPS 签名证书文件存在 /home/go/bin/ 路径下

## 1. 使用 certmagic 库获取 HTTPS证书, 生产模式( 另有测试模式)

```
package autotls

import (
	"github.com/mholt/certmagic"
	"golang.org/x/xerrors"
)

// LetsEncryptTLS the management of let's Encrypt to get domain's key and cert file
// path to save the cache of key/cert file
// email is your email , to manage domain in let's Encrypt
// domainName , like tsingson.io / www.tsingson.io ... the domain name list
func LetsEncryptTLS(path string, email string, domainName ...string) (err error) {
	config := certmagic.Config{
		Agreed:  true,
		Storage: &certmagic.FileStorage{Path: path},
		CA:      certmagic.LetsEncryptProductionCA, //  use certmagic.LetsEncryptStagingCA for testing
		Email:   email,                             // your email to management let's Encrypt
	}

	cache := certmagic.NewCache(certmagic.CacheOptions{
		GetConfigForCert: func(cert certmagic.Certificate) (certmagic.Config, error) {
			return config, nil
		},
	})

	magic := certmagic.New(cache, config)

	if len(domainName) == 0 {
		return xerrors.New("Need one DomainName at least")
	}
	err = magic.Manage(domainName)
	if err != nil {
		return
	}

	return
}
```

调用方式, 见代码

```

go autotls.LetsEncryptTLS("/home/go/bin/", "tsingson@me.com", "www.tsingson.io", "tsingson.io")

time.Sleep(1 * time.Minute) // 等待一下, 让 HTTPS 证书缓存成功........

```


首次运行前, 请创建 /home/go/bin 路径

运行时会打印日志, 约有 3秒左右, 会成功获取 HTTPS 签名证书, 存在以下路径( 注: **生产模式** )




```

/home/go/bin/acme #  tree .
.
└── acme-v02.api.letsencrypt.org
    ├── challenge_tokens
    ├── sites
    │   ├── tsingson.io
    │   │   ├── tsingson.io.crt
    │   │   ├── tsingson.io.json
    │   │   └── tsingson.io.key
    │   └── www.tsingson.io
    │       ├── www.tsingson.io.crt
    │       ├── www.tsingson.io.json
    │       └── www.tsingson.io.key
    └── users
        └── tsingson@me.com
            ├── tsingson.json
            └── tsingson.key

7 directories, 8 files

```
 

certmagic 的 Manage 将来会自动更新 HTTPS 证书


> 注: 测试模式, 缓存的 HTTPS 证书存储路径不太一样, 请自行检查路径
 


## 2. 在 net/http 上使用

```
package main

import (
    // "fmt"
    // "io"
    "net/http"
    "log"
    
    "github.com/tsingson/vk/fast/autotls"
)

func HelloServer(w http.ResponseWriter, req *http.Request) {
    w.Header().Set("Content-Type", "text/plain")
    w.Write([]byte("This is an example server.\n"))
    // fmt.Fprintf(w, "This is an example server.\n")
    // io.WriteString(w, "This is an example server.\n")
}

func main() {

	go autotls.LetsEncryptTLS("/home/go/bin/", "tsingson@me.com", "www.tsingson.io", "tsingson.io")
	time.Sleep(1 * time.Minute)
	
    http.HandleFunc("/hello", HelloServer)
    err := http.ListenAndServeTLS(":443", "/home/go/bin/acme-v02.api.letsencrypt.org/site/tsingson.io/tsingson.io.crt", "/home/go/bin/acme-v02.api.letsencrypt.org/site/tsingson.io/tsingson.io.key", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}

```

## 3. 在 fasthttp 上使用

```
package main

import (
	"log"

	"github.com/valyala/fasthttp"
	"github.com/tsingson/vk/fast/autotls"
)

func main() {

	go autotls.LetsEncryptTLS("/home/go/bin/", "tsingson@me.com", "www.tsingson.io", "tsingson.io")
	time.Sleep(1 * time.Minute)
	
	fs := &fasthttp.FS{
		// Path to directory to serve.
		Root: "/home/www/static-site",

		// Generate index pages if client requests directory contents.
		GenerateIndexPages: true,

		// Enable transparent compression to save network traffic.
		Compress: true,
	}

	// Create request handler for serving static files.
	h := fs.NewRequestHandler()

	s := &fasthttp.Server{
		Handler: h,
	}

	// Start the server.
	if err := fasthttp.ListenAndServeTLS(":443", "/home/go/bin/acme-v02.api.letsencrypt.org/site/tsingson.io/tsingson.io.crt", "/home/go/bin/acme-v02.api.letsencrypt.org/site/tsingson.io/tsingson.io.key", h); err != nil {
		log.Fatalf("error in ListenAndServe: %s", err)
	}
}

```

## 4. 小结


浏览器上验证, 步骤省略......... 


Done.   完美.......


> let's Encrypt 的测试模式, 在浏览器上验证, 会提示, 不是安全的 HTTPS 签名证书.
> 
> 换成生产模式就好了...........





_



_


_





### 关于我

网名 tsingson (三明智, 江湖人称3爷)

原 ustarcom IPTV/OTT 事业部播控产品线技术架构湿/解决方案工程湿角色(8年), 自由职业者,

喜欢音乐(口琴,是第三/四/五届广东国际口琴嘉年华的主策划人之一), 摄影与越野, 

喜欢 golang 语言 (商用项目中主要用 postgres + golang )  


_

_
 [tsingson](https://github.com/tsingson) 写于中国深圳 [小罗号口琴音乐中心](https://tsingson.github.io/music/about-studio/),   2019/08/07



