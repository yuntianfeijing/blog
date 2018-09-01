---
title: go版本管理工具gopm
date: 2018-05-14 15:44:22
tags: go
---
基于七牛云存储的用于提供版本化缓存和分发Go语言包服务的服务器。下载Go语言包从此告别低速，告别翻墙！


在国内采用go get有时会下载不到一些网站如golang.org的依赖包。可以采用gopm从golang.org一些镜像网站上下载。
   1、安装gopm 执行该命令的用户需要设置GOPATH参数
      　`go get -u github.com/gpmgo/gopm`
   2、gopm get 如果不携带-g采用，会把依赖包下载.vendor目录下面。采用-g 参数，可以把依赖包下载到GOPATH目录中
      　`gopm get -g golang.org/x/net`
