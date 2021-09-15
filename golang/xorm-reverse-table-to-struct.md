---
title: xorm命令说明
date: 2018-05-14 15:33:05
tags: go
---
xorm将数据库中的表转换成struct

安装xorm
  ```
  go get github.com/go-xorm/xorm
  go get github.com/go-xorm/cmd/xorm
  go install github.com/go-xorm/cmd/xorm
  ```
xorm命令详解：
  ```
  $:xorm help
  xorm is a database tool based xorm package.
  Version:
    0.3.0324
  Usage:
    xorm command [arguments]
    The commands are:
      reverse     reverse a db to codes
      shell       a general shell to operate all kinds of database //数据库命令行连接工具 xorm shell sqlite3 test.db
      dump        dump database all table struct's and data to standard output　//数据库导出命令　xorm dump sqlite3 test.db > test.sql
      driver      list all supported drivers　
      source      source execute std in to datasourceName //数据库导入命令　xorm source sqlite3 test.db < test.sql
  ```
reverse:
  xorm reverse command is a tool to convert your database struct to all kinds languages of structs or classes
  执行如下命令会在当前目录下生成models文件夹，此文件夹下存放databasename下所有表对应的struct文件
  `xorm reverse mysql root:passwd@(127.0.0.1:3306)/databasename?charset=utf8 $GOPATH/src/github.com/go-xorm/cmd/xorm/templates/goxorm/`
  其他数据库：
  ```
  sqlite:xorm reverse sqite3 test.db $GOPATH/src/github.com/go-xorm/cmd/xorm/templates/goxorm/
  mymysql:xorm reverse mymysql xorm_test2/root/ $GOPATH/src/github.com/go-xorm/cmd/xorm/templates/goxorm/
  postgres:xorm reverse postgres "dbname=xorm_test sslmode=disable" $GOPATH/src/github.com/go-xorm/cmd/xorm/templates/goxorm/
  ```
  reverse的Template和Config
  默认的模板和配置在$(GOPATH)/src/github.com/go-xorm/cmd/xorm/templates/goxorm/　可以自建或修改模板与配置
  Now, xorm tool supports go and c++ two languages and have go, goxorm, c++ three of default templates. In template directory, we can put a config file to control how to generating.
  ```
  lang=go
  genJson=1
  ```
  lang must be go or c++ now.genJson can be 1 or 0, if 1 then the struct will have json tag.
