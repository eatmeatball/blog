---
title:  关于用go写接口调试代码的一次记录
toc: true
date: 2021-08-02 10:09:37
# thumbnail: /images/2021/岁月的童话2.jpg
tags:
  - other
  - blog
categories:
  - other

---


开发服务端接口的时候会使用postman进行调试\测试，但是postman毕竟是一个功能比较固定的窗口软件，虽然postman也支持before after的一些操作，但是在参数复杂，或者批量调用的时候会存在操作繁琐，性能不高的情况，所以早先的时候有一些接口的调试我使用laravel-console+GuzzleHttp\Client 进行接口调试 ，postman用来辅助编写。
相比于postman这类的窗口软件，用语言直接编写接口调试的功能更加强大，并且在批量调用的时候响应上比postman快很多（postman 如果按照组进行调用，如果量大了会卡住）。后来就我就换成了go+resty。其中得益于换成了go，今天在测试环境寻找合适的测试数据，效率也比较快的完成。

<!-- more -->

我一般喜欢把同一个域名或同一个服务封装为一个client类。然后在这个client里去实现需要调试的接口调用。如下
```go

var dktForgeLoginClientIns DktForgeLoginClient

var dktForgeLoginClientInsOnce sync.Once

func DktForgeLoginClientConnection(host string) DktForgeLoginClient {
	dktForgeLoginClientInsOnce.Do(func() {
		if len(host) == 0 {
			host = "http://localhost:3080"
		}
		dktForgeLoginClientIns = DktForgeLoginClient{
			httpClient: resty.New().
				SetHostURL(host).
				SetHeader("Accept", "application/json").
				SetTimeout(time.Second * 30),
		}

	})
	return dktForgeLoginClientIns
}

func (receiver *DktForgeLoginClient) GetTempToken(userId string) (resp *resty.Response, err error) {
	return receiver.httpClient.R().
		SetQueryParams(map[string]string{
			"userId": userId,
		}).
		Get("/forgelogin/api/getDktAccessToken")
}

func (receiver *DktForgeLoginClient) template() (resp *resty.Response, err error) {
	return receiver.httpClient.R().Get("/daiketong/v1/job-position/list")
}

```

然后在 console.go 中编写展示\测试\数据提取逻辑
```go
fmt.Println("岗位列表:")
r, _ = dktClient.JobList()
showResponseJson(r, userId)

fmt.Println("创建岗位:")
for _, name := range []string{"萌德守卫"} {
    r, _ = dktClient.JobCreate(name)
    showResponseJson(r, userId)
}

jobId := util.ToInt(gjson.Get(r.String(), "data.jobId").Int())

fmt.Println("编辑岗位:")
r, _ = dktClient.JobEdit(jobId)
showResponseJson(r, userId)

fmt.Println("岗位详情:")
r, _ = dktClient.JobDetail(jobId)
showResponseJson(r, userId)
```

得益于编码的灵活性，可以随意调控请求顺序，变量存取，（postman也可以存取变量，但是相对编程来说较为繁琐,尤其是接口的调用顺序，控制起来成本比较高）。

接着是今天调试一个老接口的。背景是将底层的数据源从db换成了scf，单元测试已经可以跑起来，但是还未用接口进行请求。因为业务是刚刚接触不久，所以并不知道测试环境用什么数据可以请求这个接口。虽然可以看到这个接口的参数，但是不同的用户权限不一样。而且本次底层改动数量非少量，如果每个改动至少模拟在开发阶段至少查看一个外部接口，那么查看逻辑，再去数据库查数据寻找合适的用户数据用来进行http的参数构造那么是非常的繁琐的。由于数量较多，时间成本就变得比较大了。所以这里用了一个较为偷懒的方式，从数据库查询出一批可用的用户，从登陆开始调用，保存token，然后调用目标接口，check返回数据，可以很快search出合适的用户用于接下来针对接口的自测。

这边业务程序的逻辑是
```go
// 伪登陆client
// 获取伪登陆令牌
// authclient
// 获取真正的业务令牌
// 业务client set令牌
// 请求对应业务接口
// checkResponse
```

其中业务client由于存在登陆态，所以 业务client 我没有使用单例。

接下来就是主程序里的协程开发了，这里启动了20个协程，用redis队列进行userId的分发。其中token在获取成功后会保存到本地的redis里。
```go
func ajkAction() {
	userIdList := strings.Split(getAjkUserIdCanManageEmployee(), `,`)

	var ctx = context.Background()

	lPushKey := "workDktUserList"
	db.RedisIns().Del(ctx, lPushKey)
	db.RedisIns().LPush(ctx, lPushKey)
	db.RedisIns().LPush(ctx, lPushKey, userIdList)

	var wg sync.WaitGroup
	workNum := 20
	// 之前由于把i设置为0开始，并且设置了add20导致可能会减到负一导致报错
	for i := 1; i <= workNum; i++ {
		// 严谨点这样就不会受计算影响
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				userId, err := db.RedisIns().LPop(ctx, lPushKey).Result()
				if err != nil {
					fmt.Println(err)
					return
				}
				fmt.Println(len(userId))
				ajkClientRunner(userId)
			}
		}()
	}
	wg.Wait()
}
```
事实证明，不到1分钟就在2000个userid中筛选出可以使用的userId。
只需要在日志中把check成功的userId留下来。就可以用来接下来的自测了。由于开发过程中，每完成一个接口就实现一个对应的client-method。并且维护好主要调试可以提高调试和接口自测的效率。以及快速定位问题

```log
UserAllocatedList:
读取本地token成功
本地token尚未过期，可继续使用
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImp0aSI6ImF1dGhfdG9rZW5fcGFpcl82MTAzYTJkYzQ1MmY2MC41ODMxMzU0MCJ9.eyJpc3MiOiJodHRwczpcL1wvZGFpa2V0b25nLjU4LmNvbSIsImF1ZCI6Imh0dHBzOlwvXC9kYWlrZXRvbmcuNTguY29tIiwianRpIjoiYXV0aF90b2tlbl9wYWlyXzYxMDNhMmRjNDUyZjYwLjU4MzEzNTQwIiwiaWF0IjoxNjI4OTI0MjUyLCJleHAiOjE2Mjg5MjQyNTIsInVzZXJfaWQiOjU4NjI1fQ.rhR2YlTTcT93PFbFp3c0bqIP9kJxDU-67KnysHccgaJ8vnf7Du7Xgy3hiZ0MNZyTup9E1M7vzo5MwNVjI0V2xYh5RTu8aFjiQ6nfVab80vvIRJ3yA7l-nXQ23FqLCktHGyTjvxEm046bS4Zmt8Kldmh_lX0SritdGzqN-YiP9LTi1DCflWTAvMSnjAqRt5djFObGz0RCnCoLS93u7mVluEAML6o-zO2_-XtY1HD8_EWsHXW03NVbvu9b2mLwEA2HgFxf5jBbCP1n7V7-qyp9VHsajGDq5dWuIXz8XTa4_deVNyOLWDxRCypFOHhypAuIm2yBh0quJb4HGgkAvtUW5g
UserAllocatedList:
读取本地token成功
本地token尚未过期，可继续使用
eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImp0aSI6ImF1dGhfdG9rZW5fcGFpcl82MTAzYTJkYzI0MDhkMC43ODE4ODg4NyJ9.eyJpc3MiOiJodHRwczpcL1wvZGFpa2V0b25nLjU4LmNvbSIsImF1ZCI6Imh0dHBzOlwvXC9kYWlrZXRvbmcuNTguY29tIiwianRpIjoiYXV0aF90b2tlbl9wYWlyXzYxMDNhMmRjMjQwOGQwLjc4MTg4ODg3IiwiaWF0IjoxNjI4OTI0MjUyLCJleHAiOjE2Mjg5MjQyNTIsInVzZXJfaWQiOjU4NjI2fQ.SGve5sph9pfpF7bclNRSqXTLNHkIubJiShzSTt4TU3UfHm4vD3tYKBHc3AIAxLq92WPQOfNqJznsyQo3CZoawzejAKKolC4ym28peJV-GFdV04T7HsIMlIgpmvjpiyRhnvCoAvVCOy15-15U04U2CJtsI7xcglk6iF4BdnZZA8776oFqd7BVfgJ5gKbU45Q18RxVGUyUCvkOE33TgpCwE6uzbHwDkrs0zWZWZgavBqLhk-C5lJlrqBucs9H6vPU4NR6KqVmY_qhNV_7byL5739lladXZ5GNE3WIdbO6mYL-c_BtHUvfawgB70SdyvO2AZ1GedJ-qwWg6K9x_1rJFaQ
UserAllocatedList:
[18221] {"data":{},"errcode":"系统错误","message":"非法的管辖城市","status":"error"}
redis: nil
[18149] {"data":{"list":[{"avatar":"","cellphone":"15521577326","city_name":"上海","forward":1,"job_id":10,"job_name":"项目运营专员","job_number":null,"user_id":13585,"username":null}]},"errcode":"","message":"","status":"ok"}
redis: nil
[18163] {"data":{},"errcode":"系统错误","message":"非法的管辖城市","status":"error"}
redis: nil
```

