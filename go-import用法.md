---
title: go import用法
date: 2018-04-13 17:35:10
tags: go
---
go import的用法

`import "fmt"` 最常用的一种形式
`import "./test"`导入同一目录下test包中的内容
`import f "fmt"` 定义包的别名　可访问　`f.Printf("change package name")`
`import . "fmt"`，将fmt启用别名"."，这样就可以直接使用其内容，而不用再添加fmt，如`fmt.Println`可以直接写成`Println`

当包含多个包时可如下简写
```
import (
	"fmt"
	"errors"
)
```

我们会经常看到如下的在包引用之前加一个`_ `,意思是加载时，只执行包内的`init()`函数，不引入其他函数，也不能直接调用包内的函数
```
import (
	"database/sql"
	_ "github.com/go-sql-driver/mysql"
)
```
