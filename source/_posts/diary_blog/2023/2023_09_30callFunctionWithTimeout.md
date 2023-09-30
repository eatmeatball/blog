---
title: callFunctionWithTimeout
toc: true
date: 2023-09-30 09:18:41
tags:
  - other
  - blog
categories:
  - other

---



```go 

func callFunctionWithTimeout[R any, T func() R](f T, timeout time.Duration, defaultValue R) R {
	var result R
	var mu sync.Mutex
	done := make(chan bool, 1)

	go func() {
		result = f()
		mu.Lock()
		defer mu.Unlock()
		done <- true
	}()

	select {
	case <-done:
		return result
	case <-time.After(timeout):
		mu.Lock()
		defer mu.Unlock()
		return defaultValue
	}
}

```

<!--more-->


