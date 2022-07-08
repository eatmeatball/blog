---
title: go:我的gin ～控制器也能跑出panic了
toc: true
date: 2022-07-08 09:16:51
tags:
  - other
  - blog
categories:
  - other

---

前两天写了写了一篇关于go无法捕获panic的的情况。当时并没有想太多，今天突然想到如果我在 gin 的控制器函数中使用了 `go` 并抛出了 `panic` 那么会不会被捕获呢？。代码如下

```go
func About(c *gin.Context) {
	go func() {
		time.Sleep(time.Second * 10)
		panic("嘿嘿")
	}()
	c.JSON(http.StatusOK, gin.H{
		"message": "Hello~ Now you see a json from gin",
	})
}
```

结论是请求后10s后就抛出了panic。gin 之所以可以捕获 action 中直接跑出的 `panic` 是因为 gin 采用的洋葱设计，在外层有着对 panic 的捕获操作。 不同的是，由 go 运行并抛出的 panic 是不会被捕获的。对于这种情况，可以用高阶高数处理一下。不过 子协程 的影响还是比较大的。比如引入的包开了一个 `goroutine` 但是并没有处理内部的 `panic` 那么就会让程序产生无法控制的中断。 高阶函数的部分如下

<!--more-->



```go

func About(c *gin.Context) {
	go safeGoRoutine(func() {
		time.Sleep(time.Second * 10)
		panic("嘿嘿")
	})
	c.JSON(http.StatusOK, gin.H{
		"message": "Hello~ Now you see a json from gin",
	})
}

func safeGoRoutine(a func()) {
	defer func() {
		if err := recover(); err != nil {
			fmt.Println(err)
		}
	}()
	a()
}
```