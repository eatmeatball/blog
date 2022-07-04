---
title: 'go:channel,queue,进程管理,关闭channel'
toc: true
date: 2022-07-04 17:13:07
tags:
  - other
  - blog
  - go
categories:
  - other

---

今天看到一篇有关`channel`的问答。文章中中提到了`channel`的缓存区，当时我看到缓存区的反应是 是不是可以把我之前写的队列用 `channel`
进行替换。随着 `channel` 的研究，发现水很深。相对于`不要通过共享内存来通信，要通过通信来共享内存`这句看似简单的话，使用起来存在的问题还是不少的。

<!--more-->


# 一段有问题的 channel 代码

这是一段我曾经写的 channel 使用代码存在有瑕疵。因为使用的是无缓冲通道，所以不容易发现。

```go
func main(){
  cInt := make(chan int, 0) // 无缓冲通道
	endInt := make(chan int)

	go func() {
		for i := 1; i <= 10; i++ {
			fmt.Println(i, "wait into channel")
			cInt <- i
		}
		// close(cInt)
		endInt <- 1
	}()

	write := func() {
		for {
			select {
			case data := <-cInt:
				fmt.Println("get ", data, " from channel")
				time.Sleep(time.Millisecond * 500)
				break
			case <-endInt:
				return
			}
		}
	}

	write()
}
```

如果把缓冲区变得比较大，那么问题就凸显出来了

```go
cInt := make(chan int, 10)
```

`write()` 提前结束了，并没有坚持到最后。

```sh
···
9 wait into channel
10 wait into channel
get  1  from channel
get  2  from channel
get  3  from channel
# end
```

原因就是生产端同时生产 `cInt` ,`endInt` 。接受方希望通过 `endInt` 判断结束，但是 select 的消费顺序是无序的。也就是说在  `cInt` ,`endInt` 均有数据的时候，并不是优先消费`cInt`。一旦消费`endInt` 就跳出了整体，从而抛弃了还未处理的数据。  
  
当前场景的问题修复是比较简单的，可以在生产端在数据生产完毕后 `close` 这个 `channel`， 对于消费方可以改用以下带啊吗进行退出操作。

```go
for data := range cInt {
  fmt.Println(data)
}
```

`range` 会在所有数据读取完毕后跳出循环。现在已经修复完毕了，那么可以在这个基础上使用 `channel` 作为一个任务分发的队列呢。比如爬虫的后续任务的入队，和当前任务的分发。我的观点是，也不是不可以，但是有局限性。可以看一下下面的代码，受限于 `channel` 长度的影响，下述代码会发生死锁。

```go
func channelSpider(cmd *cobra.Command, args []string) {
	cInt := make(chan int, 10)
	// 模拟爬取的数据
	cInt <- 1
	spider := func(i int) {
		// todo spider
		// push next job
		nestJobNum := rand.Intn(3)
		for i := 0; i <= nestJobNum; i++ {
			cInt <- i
		}
	}
	for i := 1; i <= 3; i++ {
		go func() {
			for cData := range cInt {
				fmt.Println("开始消费")
				spider(cData)
				fmt.Println("消费结束")
			}
		}()
	}
	time.Sleep(time.Second * 100)
}
```


# 其他

对于上述的场景和使用的方法，就又带来了若干个问题。

## 开辟大量 chan 不进行关闭是否会导致内存泄漏？

内存是否泄露我们可以进行一下测试。观察输出结果很显然没有`close`也是可以被gc的。

```go
// 无关闭无泄漏，会回收
func goManyChannelTest(cmd *cobra.Command, args []string) {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("%d B\n", m.Alloc/8)
	var i int64

	for i = 0; i < 100000000000; i++ {
		useChan()

		if i%1000000 == 0 {
			runtime.ReadMemStats(&m)
			fmt.Printf("Thousand-hand:useMem %d KB\n", m.Alloc/1024/8)
		}
	}
}
func useChan() {
	cInt := make(chan int, 0)
	//defer close(cInt)
	//fmt.Println(cInt)
	if false {
		fmt.Println(cInt)
	}
}
```

## 如何判断一个 chan 是否关闭


下面只是其中一个方法，不是很推荐，仅仅是无数据情况下生效，并且在有数据的时候可能会导致数据被误取，用 `for .. range ` 也有同样的问题

```go 
isClose := func(a chan int) bool {
  select {
  case <-a:
    return true
  default:
    return false
  }
}
```

## 场景1问题:`任务分发是否适合使用 channel`

可以用，个人不是很推荐，可以去实现一个真实的队列，如果可以接受定长队列的话，也可以用 `channel`。

```go

var mutex = &sync.Mutex{}

var queueList = make(map[string][]string)

func QueueRPush(key string, data ...string) {
	mutex.Lock()
	defer mutex.Unlock()
	queue, _ := queueList[key]
	queue = append(queue, data...)
	queueList[key] = queue
}

func QueueLPop(key string) (string, error) {
	mutex.Lock()
	defer mutex.Unlock()
	queue, _ := queueList[key]
	if len(queue) > 0 {
		result := queue[0]
		queue = queue[1:]
		queueList[key] = queue
		return result, nil
	}
	return "", errors.New("Queue is null")
}

func QueueLen(key string) int {
	mutex.Lock()
	defer mutex.Unlock()
	queue, ok := queueList[key]
	if ok {

		return len(queue)
	}
	return 0
}
```

## 场景2问题:`多任务执行用 channel 判断所有任务结束是否合适`

可以用，也可以用 `wg` 去实现，个人认为两者并无优劣

```go
type workJob struct {
	done chan bool
}

// 可以为了实现功能而使用 channel ，不要为了使用 channel 而实现
func channelManagerMoreGoFinish(cmd *cobra.Command, args []string) {
	wList := []workJob{}
	for i := 1; i <= 10; i++ {
		// 如果不初始化会阻塞
		newJob := workJob{make(chan bool)}
		go func(job workJob, i int) {
			t := rand.Int31n(3)
			fmt.Println("will sleep ", t, "second")
			time.Sleep(time.Second * cast.ToDuration(t))
			fmt.Println(i, "job will end")
			job.done <- true
			fmt.Println(i, "job end")
		}(newJob, i)
		wList = append(wList, newJob)
	}

	for _, workJobItem := range wList {
		<-workJobItem.done
	}
	fmt.Println("all end")
}

```

个人推荐使用 `sync.WaitGroup` 进行控制，可读性更强。更容易理解

```go
// Together 并行执行
func Together(job func(goId int), counter int) {
	var wg sync.WaitGroup
	for i := 0; i <= counter; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			job(i)
		}(i)
	}
	wg.Wait()
}

```