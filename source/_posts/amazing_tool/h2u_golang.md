---
title: how to use golang
toc: true
date: 2021-01-27 10:02:08
thumbnail: https://cdn.jsdelivr.net/gh/removeif/blog_image/img/2019/20190919221611.png
tags:
  - other
  - blog
  - go
categories:
  - HowToUse
---

<!-- TOC -->

- [go环境设置](#go环境设置)
	- [go get 代理设置](#go-get-代理设置)
	- [go mod](#go-mod)
	- [go env](#go-env)
	- [关于下载库和下载软件](#关于下载库和下载软件)
- [基础](#基础)
- [包推荐](#包推荐)
- [练习](#练习)
- [闭包练习](#闭包练习)
- [gin](#gin)
- [gorm](#gorm)
	- [gorm 分表写法](#gorm-分表写法)
- [model 开发](#model-开发)
- [cgo](#cgo)
- [取苹果](#取苹果)

<!-- /TOC -->

<!--more-->

# go环境设置

## go get 代理设置

- 对应的shell代理设置，cmd/powershell
- git代理设置

```
set http_proxy=http://127.0.0.1:1081
set https_proxy=http://127.0.0.1:1081
$ENV:http_proxy="http://127.0.0.1:1081"
$ENV:https_proxy="http://127.0.0.1:1081"
git config --global http.proxy http://127.0.0.1:1081
git config --global https.proxy http://127.0.0.1:1081
git config --global --unset http.proxy
git config --global --unset https.proxy
```

## go mod 
```
go mod download  //下载依赖包
go mod edit  //工具或脚本编辑go.mod
go mod graph  //打印模块依赖图
go mod init  //在当前目录初始化mod
go mod tidy  //拉取缺少的模块，移除不用的模块。
go mod vendor  //将依赖复制到vendor下
go mod verify  //验证依赖是否正确
go mod why  //解释为什么需要依赖
go list -m -json all  //依赖详情
```
go mod 练习
```
首先
mkdir go_t && cd go_t
go mod init app
go get -u github.com/codesenberg/bombardier // 这是一个http压测客户端，
cat go.sum // go.sum 安装了没有用的东西，因为打算全局安装，但是并不想要引入当前项目
go mod tidy // 无效 因为当前目录没有impact 任何go的包
// 创建一个go文件
go mod tidy  
cat go.sum
// 不会影响全局
go install  github.com/codesenberg/bombardier@latest 

```

## go env

go 1.13提供了 go env -w命令来写环境变量
```
go env -w XXX="XXX"
go env -u XXX

go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```


win: `用户\AppData\Roaming\go\env`
linux: ` /root/.config/go/env`

## 关于下载库和下载软件

> 以下基于 go 1.17 


如果你要下载一个由`go`编写的的可执行程序，你应该执行 `go install xxxx`。这会在 `GO_PATH/bin/` 下安装可执行文件。
如果你要引用第三方库进行代码开发，你可以使用 `go get -d ` （仅下载代码不进行编译）。

可以认为目前是 go 1.17 还是一个过渡时期，未来 `go get`应该不再进行编译操作，`-d` 会演变为默认参数。而 `go install`则进化为下载可执行软件的专用方法。

目前 go 1.17 用 go get 还是可以下载+编译的。未来应该是不能这么操作了

# 基础

# 包推荐

# 练习

# 闭包练习
```go
package main 
func add(x1, x2 int) func(x3 int, x4 int) (int, int, int) {
	i := 0
	return func(x3 int, x4 int) (int, int, int) {
		i++
		return i, x1 + x2, x3 + x4
	}
}
```

# gin

# gorm

## gorm 分表写法

假设 `user` 按照 `uid` 分表。
**关键点**：设置`db.Table`，同时在查询前调用`Scopes`，因为`Scopes`传递的是闭包，所以可以编写一个构建闭包的函数进行设置。

```go

type KUser struct {
	ID              uint32    `gorm:"primaryKey;column:id;type:int(10) unsigned;not null"`              // 用户表自增id
	UId             uint64    `gorm:"unique;column:uid;type:bigint(20) unsigned;not null"`              // 用户中心passId
	NickName        string    `gorm:"column:nick_name;type:varchar(255);not null;default:''"`           // 用户昵称
}

func (u KUser) getSuffixByUid() uint64 {
	return u.UId % uint64(32)
}

type kUserRepository struct {
}

var kur *kUserRepository

func init() {
	kur = new(kUserRepository)
}

func setGetSuffix(suffix int) func(db *gorm.DB) *gorm.DB {
	return func(db *gorm.DB) *gorm.DB {
		return db.Table("user_" + types.ToString(suffix))
	}
}

func setGetSuffixByUid(uid int64) func(db *gorm.DB) *gorm.DB {
	return setGetSuffix(getSuffixByUid(uid))
}

func getSuffixByUid(uid int64) int {
	return types.ToInt(uid % 32)
}

func (kur kUserRepository) FindBySuffix(suffix int) []KUser {
	var kUsers []KUser
	r := dbPool.GetDB().Scopes(setGetSuffix(suffix)).Find(&kUsers)
	fmt.Println(r.Error)
	return kUsers
}

func GetKUserRepository() *kUserRepository {
	return kur
}

```

# model 开发

go 模块的开发并不复杂，但是其为了兼容老版本，所采取的策略比较有意思。相比与 `composer`、`npm`  这种每个包指定版本的做法。go mod 可以在一个项目下依赖同一个库的不同版本的代码。

首先
```
mkdir hello-go-mod
cd hello-go-mod
git init 
go mod init github.com/eatmeatball/hello-go-mod
mkdir hello
cd hello
echo "package hello


func Hello() string{
	return "hello"
}" > function.go

cd ..
git add .
git commit -m 'code a lib for go'
git push
```

当推送完成之后你就可以使用

```
go get -u  github.com/eatmeatball/hello-go-mod
```

在其他项目拉取该项目了。默认为`v0.0.0`。  

接下来就是不一样的地方了。你可以在这个项目上打`tag`，当`tag`为 `v1.x.x`/`v0.x.x` 拉取是没有任何问题的。  

当你把 `tag` 打到 `v2.x.x`。你会发现无法拉取  

拉取代码的时候会出现
```
go get: github.com/eatmeatball/hello@v2.0.1: reading https://goproxy.cn/github.com/eatmeatball/hello/@v/v2.0.1.info: 404 Not Found
        server response: not found: github.com/eatmeatball/hello@v2.0.1: invalid version: module contains a go.mod file, so major version must be compatible: should be v0 or v1, not v2
```

是不是满脑子充满了问号？  

这里有一篇官方的blog[v2-go-modules](https://blog.golang.org/v2-go-modules)。官网的内容中定义了一个原则
> 如果旧包和新包的导入路径相同，新包必须向后兼容旧的包
对比以下之前用过的包管理，是不是有点不一样？比如 php 的 composer ，里面的高版本都是为了进行破坏性的升级才修改的。升级高版本是不是要修改代码？虽然go中高版本也可以写api破坏性的代码，但是其高于v2的版本通过`go get `后本地位置是不一样的。所以在go的项目中，你可以在一个项目中同时引入一个库的`v1`,`v2`,`v3`代码进行编译。这里如果你想要发布非`v0`、`v1`的版本你需要把`go.mod`文件修改

修改前
```go.mod
module github.com/eatmeatball/hello-go-mod

go 1.16
```

修改后
```go.mod
module github.com/eatmeatball/hello-go-mod/v3

go 1.16
```
这样，即使你发布的新版本存在破坏性，但是拉取的时候go按照顶级版本划分了目录，就可以使其同时使用高低版本代码

> [https://github.com/rsc/quote/tree/master](https://github.com/rsc/quote/tree/master)

根据官网的demo你可以一个项目多个目录，也可以一个项目多个分支来控制版本。



# cgo



> GCC原名为GNU C语言编译器（GNU C Compiler），只能处理C语言。但其很快扩展，变得可处理C++，后来又扩展为能够支持更多编程语言，如Fortran、Pascal、Objective -C、Java、Ada、Go以及各类处理器架构上的汇编语言等，所以改名GNU编译器套件

MinGW-w64下载地址：[https://sourceforge.net/projects/mingw-w64/files/](https://sourceforge.net/projects/mingw-w64/files/)

选择合适的版本

- i686纯32位版供32位win系统使用
- x86_64是64位系统用的版本
- seh结尾是纯64位编译
- sjlj结尾是32 64两种编译，需加-m32或-m64参数
- posix通常用于跨平台，比win32兼容性好一些

最终下载版本`x86_64-posix-sjlj`

解压后bin目录加入到环境变量


# 取苹果



```go
package main

import (
	"fmt"
	"time"
)

func  main() {

	fmt.Println("this is a demoC")

	plate := make(chan int)
	apple := make(chan int)
	orange := make(chan int)
	go func() {
		for {
			time.Sleep(time.Second)
			dadPlate := <-plate
			fmt.Println("盘子可以放入一个水果 dad Plate", dadPlate)
			apple <- 1
			fmt.Println("dad放入一个苹果")
		}
	}()

	go func() {
		for {
			time.Sleep(time.Second)
			momPlate := <-plate
			fmt.Println("盘子可以放入一个水果mom Plate", momPlate)
			orange <- 1
			fmt.Println("mom放入一个橘子")
		}

	}()

	go func() {
		for {
			time.Sleep(time.Second)
			sonOrange := <-orange
			fmt.Println("儿子取走一个橘子 sonOrange Plate", sonOrange)
			plate <- 1
			fmt.Println("儿子清空盘子")
		}

	}()
	go func() {
		for {
			time.Sleep(time.Second)
			daughterApple := <-apple
			fmt.Println("女儿取走一个苹果 daughterApple Plate", daughterApple)
			plate <- 1
			fmt.Println("女儿清空盘子")
		}
	}()
	plate <- 1
	for {
		time.Sleep(time.Second)
	}
}
```