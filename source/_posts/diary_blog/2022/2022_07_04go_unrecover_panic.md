---
title: 'go:为什么我的 panic 逃跑了'
toc: true
date: 2022-07-04 17:24:47
tags:
  - other
  - blog
categories:
  - other

---

```go
func action(){
  defer func() {
		if r := recover(); r != nil {
			log.Println("get panic",r)
		}
	}()
  go func(){
    time.Sleep(time.Second*10)
    panic(1)
  }()
  fmt.Println("end")
}
```

可以这样理解，`go`的代码和 action 已经是两个世界的东西了。所以即使在主进程也无法捕获。从另一个方面来说，这个方法的 defer 早就已经执行结束了。在`fmt.Println("end")`后，就已经执行。所以如果要捕获这个 panic 要在他的父函数或者本函数，而不是主进程。

<!--more-->


