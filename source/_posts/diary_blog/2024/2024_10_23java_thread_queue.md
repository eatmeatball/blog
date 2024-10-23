---
title: java_thread_queue 任务不退出问题
toc: true
date: 2024-10-23 15:03:19
tags:
  - other
  - blog
categories:
  - other

---


之前一直用go写的多消费者，因为go存在channel，所以可以用有缓channel模拟队列进行消费和退出，但是java不存在channel，所以要是用一些内存queue，结果由于退出条件的判断不一样，导致java的代码进入空等待。一只没有释放资源。

go 原始代码如下
```go

func main(){

	type Item struct {
		value string
	}

	queue:=  make(chan Item,100)
	task := StartTogether(func(){
		for item:= range queue{
			fmt.Println(item)
		}
	},10)
	for i:=1;i<=10000;i++{
		queue<-Item{
			value: fmt.Sprintf("%v",i),
		}
	}
	
	close(queue)
	task.Wait()
}
func StartTogether(job func(), counter int) *sync.WaitGroup {
	var wg sync.WaitGroup
	for i := 1; i <= counter; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			job()
		}()
	}
	return &wg
}

```


<!--more-->

一开始改造的java代码如下,这份代码会出现偶尔无法退出的情况。最后定位到是因为 `Object item = queue.take();` 发生了阻塞。
```java
public class ThreatQueue {
    private static final BlockingQueue<Object> queue = new LinkedBlockingQueue<>(2000);

    private static final ExecutorService executor = Executors.newFixedThreadPool(10);

    private static final AtomicBoolean done = new AtomicBoolean(false);

    public static void main(String[] args) throws InterruptedException {
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 2; i++) {
                    queue.put(i);
                }
                done.set(true);
                System.out.println("Produced all items");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        producer.start();
        for (int i = 1; i <= 10; i++) {
            executor.submit(() -> {
                try {
                    while (!done.get() || !queue.isEmpty()) {
                        Object item = queue.take();
                        System.out.println(Thread.currentThread().getName() + "Consumed: " + item);
                    }
                    System.out.println("Consumed all items");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt(); // 保持中断状态
                }


            });
        }
        producer.join();

        executor.shutdown();
        while (!executor.isTerminated()){

            System.out.println("waiting");
            Thread.sleep(100);
        }
        System.out.println("finish");
    }
}
```

和go代码不同，go代码是range在消费完channel后，如果channel已经close，则会直接退出循环。那么当前goroutine退出，并执行wg.done()。整个过程完全借助于go本身的，所以整个托管过程是比较顺畅的。

对比之下由于java版本的代码要实现上面的功能。需要自己实现queue空判断，和任务结束。但是取值是在空判断之后，这就出现在并发场景下，可能判断非空，但是取值的时候已经没有任务了。导致` Object item = queue.take();` 进入无尽的等待。从而`!executor.isTerminated()`始终不处于结束

```java
while (!done.get() || !queue.isEmpty()) {
    Object item = queue.take(); // 问题的关键在于本行代码, queue.isEmpty 进入前为false，但是进入本行后queue不在有数据，数据被其他消费者消费完毕了。

```

关于此处的代码更改，可以尝试破除queue的阻塞取值逻辑，即更换为以下代码

```java
Object item = queue.poll(1, TimeUnit.SECONDS);
if (item == null) {
    System.out.println(Thread.currentThread().getName() + " 获取失败");
    continue;
}
```
区别在于`queue.poll(1, TimeUnit.SECONDS)`不会无限制的等待获取队列中的内容，而是1秒后如果取不到那么就返回一个null，我们需要判断一下是否为null，如果为null跳出本次循环，进入`while()`判断，如果当前处于生产者不在产生数据，并且队列为就可以正常退出程序啦。