---
title: "postgresql 11 相关"
date: 2019-08-01T12:02:57+08:00
hidden: false
draft: true
author: "tsingson"
categories : [ "Development" ]
series: “postgres"
tags: [postgres]
keywords: [tsingson]

description: "postgresql 11 相关"
slug: "postgres11"


---




# postgresql 11  in RHEL 7 / centOS 7



## install and configuration 



```
pg11 需发安装的库

postgresql11-server.x86_64

postgresql11-contrib.x86_64
postgresql11-devel.x86_64
postgresql11-llvmjit.x86_64 
postgresql11-libs.x86_64 


```





```
在 主库配置文件里加

shared_preload_libraries = 'pg_stat_statements'         # (change requires restart)
jit = on                                # allow JIT compilation
jit_provider = 'llvmjit'                # JIT implementation to use

```





## reference 

1. [https://github.com/okbob/plpgsql_check](https://github.com/okbob/plpgsql_check)