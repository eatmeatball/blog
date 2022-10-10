---
title: go 升级 gin/fiber 的控制器
toc: true
date: 2022-10-10 18:24:28
tags:
  - go
  - blog
categories:
  - other
  - go

---

之前用 go 的泛型，写了相关的 collection 操作函数。所以又了对 fiber gin 这类 http lib 升级的想法，使得控制器的编写更加方便。

这里放一个最基础的例子，更多的编写放在了后方。


```go
// ginUpP  支持params 参数
func ginUpP[T any](action func(request T) component.Response) func(c *gin.Context) {
	return func(c *gin.Context) {
		var params T
		_ = c.ShouldBind(&params)
		err := validate.Struct(params)
		if err != nil {
			c.JSON(http.StatusBadRequest, component.DataMap{
				"msg": err.Error(),
			})
		}
		response := action(params)
		c.JSON(response.Code, response.Data)
	}
}
```

这段代码可以使得类似这样的控制器代码很方便的撰写,将大量的重复代码剥离出控制器外。

```go
type GetTwitterUserListParam struct {
	Page     int    `form:"page"`
	PageSize int    `form:"pageSize"`
	Search   string `form:"search"`
}

func GetTwitterUserList(param GetTwitterUserListParam) component.Response {
	pageData := FTwitterUser.Page(FTwitterUser.PageQuery{
		Page: param.Page, PageSize: param.PageSize, Search: param.Search,
	})

	return component.SuccessResponse(component.DataMap{
		"itemList": arms.ArrayMap(func(item FTwitterUser.FTwitterUser) TLink {
			return TLink{
				ScreenName: item.ScreenName,
				Name:       item.Name,
				Desc:       item.Desc,
				Url:        fmt.Sprintf("https://twitter.com/%v/with_replies", item.ScreenName),
				CreateTime: item.CreateTime.Format("2006-01-02 15:05:05"),
			}
		}, pageData.Data),
		"size":    pageData.PageSize,
		"total":   pageData.Total,
		"current": param.Page,
	})
}
```

唯一的区别就是路由注册的代码可能会稍微长一点点
```go
apiGroup.GET("/GetTwitterUserList", ginUpP(controllers.GetTwitterUserList))
```

<!--more-->


```go

import (
	"github.com/gin-gonic/gin"
	"github.com/go-playground/validator/v10"
	"github.com/gofiber/fiber/v2"
	"net/http"
	"thh/app/http/controllers/component"
)

var validate = validator.New()

// fiberUpP  支持params 参数
func fiberUpP[T any](action func(request T) component.Response) func(c *fiber.Ctx) error {
	return func(c *fiber.Ctx) error {
		var params T
		_ = c.BodyParser(&params)
		err := validate.Struct(params)
		if err != nil {
			return c.Status(http.StatusBadRequest).JSON(fiber.Map{
				"msg": err.Error(),
			})
		}
		response := action(params)
		return c.Status(response.Code).JSON(response.Data)
	}
}

// fiberUpNP  支持空参数
func fiberUpNP(action func() component.Response) func(c *fiber.Ctx) error {
	return func(c *fiber.Ctx) error {
		response := action()
		return c.Status(response.Code).JSON(response.Data)
	}
}

// fiberUpAuth  支持获取user 支持参数 在 auth 中间件后使用
func fiberUpAuth[T any](action func(ctx component.RequestContext, request T) component.Response) func(c *fiber.Ctx) error {
	return func(c *fiber.Ctx) error {
		userId := c.Locals("userId").(uint64)
		if userId == 0 {
			return c.Status(http.StatusUnauthorized).JSON(fiber.Map{
				"message": "un Login",
			})
		}
		var params T
		_ = c.BodyParser(&params)
		err := validate.Struct(params)
		if err != nil {
			return c.Status(http.StatusBadRequest).JSON(fiber.Map{
				"msg": err.Error(),
			})
		}
		response := action(component.RequestContext{
			UserId: userId,
		}, params)
		return c.Status(response.Code).JSON(response.Data)
	}
}

// fiberUpNPAuth 支持获取 user 无参数下使用
func fiberUpNPAuth(action func(ctx component.RequestContext) component.Response) func(c *fiber.Ctx) error {
	return func(c *fiber.Ctx) error {
		userId := c.Locals("userId").(uint64)
		if userId == 0 {
			return c.Status(http.StatusUnauthorized).JSON(fiber.Map{
				"message": "un Login",
			})
		}
		response := action(component.RequestContext{
			UserId: userId,
		})
		return c.Status(response.Code).JSON(response.Data)
	}
}

// ginUpP  支持params 参数
func ginUpP[T any](action func(request T) component.Response) func(c *gin.Context) {
	return func(c *gin.Context) {
		var params T
		_ = c.ShouldBind(&params)
		err := validate.Struct(params)
		if err != nil {
			c.JSON(http.StatusBadRequest, component.DataMap{
				"msg": err.Error(),
			})
		}
		response := action(params)
		c.JSON(response.Code, response.Data)
	}
}

// ginUpNP  支持空参数
func ginUpNP(action func() component.Response) func(c *gin.Context) {
	return func(c *gin.Context) {
		response := action()
		c.JSON(response.Code, response.Data)
	}
}

// ginUpAuth  支持获取user 支持参数 在 auth 中间件后使用
func ginUpAuth[T any](action func(ctx component.RequestContext, request T) component.Response) func(c *gin.Context) {
	return func(c *gin.Context) {
		userIdData, _ := c.Get("userId")
		userId := userIdData.(uint64)
		if userId == 0 {
			c.JSON(http.StatusUnauthorized, fiber.Map{
				"message": "un Login",
			})
		}
		var params T
		_ = c.ShouldBind(&params)
		err := validate.Struct(params)
		if err != nil {
			c.JSON(http.StatusBadRequest, fiber.Map{
				"msg": err.Error(),
			})
		}
		response := action(component.RequestContext{
			UserId: userId,
		}, params)
		c.JSON(response.Code, response.Data)
	}
}

// ginUpNPAuth 支持获取 user 无参数下使用
func ginUpNPAuth(action func(ctx component.RequestContext) component.Response) func(c *gin.Context) {
	return func(c *gin.Context) {
		userIdData, _ := c.Get("userId")
		userId := userIdData.(uint64)
		if userId == 0 {
			c.JSON(http.StatusUnauthorized, fiber.Map{
				"message": "un Login",
			})
		}
		response := action(component.RequestContext{
			UserId: userId,
		})
		c.JSON(response.Code, response.Data)
	}
}

```

