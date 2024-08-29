---
title: java_queue
toc: true
date: 2024-08-29 13:53:00
tags:
  - other
  - blog
categories:
  - other

---

java queue with thread


<!--more-->


```java
package com.example.jtool;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.atomic.AtomicBoolean;

public class ThreatQueueTest {
    private static final BlockingQueue<Object> queue = new LinkedBlockingQueue<>();

    private static final ExecutorService executor = Executors.newFixedThreadPool(10);

    private static volatile int count = 0;

    public static synchronized void increment() {
        count++;
    }

    public static synchronized int getCount() {
        return count;
    }

    private static final AtomicBoolean done = new AtomicBoolean(false);


    public static void main(String[] args) throws InterruptedException {
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 100; i++) {
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
