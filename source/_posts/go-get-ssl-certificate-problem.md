---
title: go get 时出现 SSL certificate problem
date: 2021-12-12 10:40:01
tags: [go,git]
---

使用 go get 下载依赖时，出现 SSL certificate problem: certificate has expired 错误<!-- more -->：  

```bash
go get gopkg.in/ini.v1                               
# cd .; git clone -- https://gopkg.in/ini.v1 /Users/xxx/go/src/gopkg.in/ini.v1
Cloning into '/Users/xxx/go/src/gopkg.in/ini.v1'...
fatal: unable to access 'https://gopkg.in/ini.v1/': SSL certificate problem: certificate has expired
package gopkg.in/ini.v1: exit status 128
```

这是该模块所在服务端使用的 https 证书问题，certificate has expired 是证书已经过期，类似的错误还有 Invalid 无效证书等。可以通过设置不进行证书验证：

```bash
git config –global http.sslVerify false
```

 另外，也可以通过查找该模块是否有 github 的托管，通过 github 的地址拉取，例如 `gopkg.in/ini.v1` 模块在 github 上的仓库为 `github.com/go-ini/ini` 。可以通过 `go get github.com/go-ini/ini`  获取。