---
title: "在CentOS 7上安装Envoy Proxy"
date: 2019-05-30T22:02:57+08:00
hidden: false
draft: true
author: "tsingson"

categories : [ "Development" ]
series: “mac"
tags: [mac,golang]
keywords: [tsingson,gdihf,music,harmonica,blues]

description: "mac 并不适合编程"
slug: "envoy-centos"

---

# 在CentOS 7上安装Envoy Proxy



https://sxi.io/how-to-install-envoy-proxy-on-centos-7

Envoy是一个开源、edge和服务代理，它将网络功能从应用程序中抽象出来，你可以将Envoy代理部署在应用程序旁边作为edge代理运行，一旦所有应用程序流量都配置为通过Envoy网格，你就可以持续控制并观察网络中发生的情况。

Envoy处理的一些网络功能包括：服务发现、指标监控、访问记录、追踪、身份验证和授权等等。

 

## 在CentOS 7上安装Envoy Proxy

在本文中，我们将使用GetEnvoy工具在CentOS 7上安装Envoy Proxy。

### 第1步：安装yum-config-manager实用程序。

通过运行命令在CentOS 7上安装yum-config-manager实用程序：

```
sudo yum install -y yum-utils
```

参考：软件包管理基础：apt,yum,dnf,pkg。

### 第2步：添加GetEnvoy存储库

使用以下命令为CentOS 7添加GetEnvoy存储库：

```
sudo yum-config-manager --add-repo https://getenvoy.io/linux/centos/tetrate-getenvoy.repo
```
如果要安装Nightly软件包，请使用-enable选项启用：
```
sudo yum-config-manager --enable tetrate-getenvoy-nightly
```
### 第3步：在CentOS 7上安装Envoy二进制文件

要从添加的存储库在CentOS 7上安装Envoy二进制文件，请运行以下命令：
```
sudo yum install -y getenvoy-envoy
```
验证CentOS 7上安装的Envoy版本，运行：

```
$ envoy --version
```



## install envoy in MacOS


Install Envoy on macOS
Requirements
To install Envoy, you need to have installed Homebrew. The package is only tested in macOS 10.14.

### Installation
Add the Tetrate Homebrew GetEnvoy tap.
```
$ brew tap tetratelabs/getenvoy
```
==> Tapping tetratelabs/getenvoy
Cloning into '/usr/local/Homebrew/Library/Taps/tetratelabs/homebrew-getenvoy'...
Tapped 1 formula.

### Install Envoy binary.
To install the nightly version instead, add --HEAD flag to the install command.
```
$ brew install envoy
```
==> Installing envoy from tetratelabs/getenvoy
==> Downloading ...
######################################################################## 100.0%
🍺  /usr/local/Cellar/envoy/1.10.0: 3 files, 27.9MB, built in 13 seconds

### Verify Envoy is installed.
```
$ envoy --version
```