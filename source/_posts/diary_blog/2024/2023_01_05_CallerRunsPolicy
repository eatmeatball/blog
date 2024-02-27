---
title: CallerRunsPolicy
toc: true 
date: 2024-02-27 09:47:23 
tags:
  - other
  - blog
  - queue
  - task
categories:
  - other
---



<!--more-->

```java
    private ThreadPoolExecutor readExecutor  = new ThreadPoolExecutor(2, 2, 10, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(100), new ThreadPoolExecutor.CallerRunsPolicy());

    public static void main(String[] args) {
        new ThreatTest().deal();
        System.exit(0);
    }

    public void deal() {


        for (int i = 1; i <= 10000; i++) {
            // 门店公司不玩跳公
            int finalI = i;
            readExecutor.execute(() -> execute(finalI));
            System.out.println("run" + i);
        }

        while (true) {
            if (writeExecutor.isTerminated() && readExecutor.isTerminated()) {

                break;
            }
        }
    }

    private void execute(int i) {
        try {
            Thread.sleep(1000);
            System.out.println(i);
        } catch (InterruptedException e) {

        }
        return;
    }
```


```go

func taskQueueJob(id int, wg *sync.WaitGroup, ch chan struct{}, maxChan chan struct{}) {
	defer func() {
		<-ch // 任务完成后从通道中取出一个元素
		<-maxChan
		wg.Done()
	}()
	task(id)

}
func task(id int) {
	fmt.Printf("Task %d is running...\n", id)
	time.Sleep(1 * time.Second)
	fmt.Printf("Task %d has been completed\n", id)
}

func runCallerRunsPolicy(cmd *cobra.Command, args []string) {
	taskQueue := make(chan struct{}, 10)
	maxTask := make(chan struct{}, 2)
	wg := sync.WaitGroup{}
	for i := 1; i <= 15; i++ {
		select {
		case taskQueue <- struct{}{}:
			maxTask <- struct{}{}
			wg.Add(1)
			go taskQueueJob(i, &wg, taskQueue, maxTask)
		default:
			fmt.Printf("Task %d is executed by the main process\n", i)
			task(i)
		}
	}

	close(taskQueue)
	wg.Wait()
	close(maxTask)

}

```
