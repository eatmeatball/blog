---
title: 'go collection'
toc: true
date: 2022-09-05 20:28:21
tags:
  - other
  - blog
categories:
  - other

---

- [基于接口实现map泛型遍历](#基于接口实现map泛型遍历)
- [一些方便的集合转换操作](#一些方便的集合转换操作)

<!--more-->

# 基于接口实现map泛型遍历

> map 的key必须是可以进行比较的，如果之后需要扩展可以进行组合

```go
type OrderedType interface {
    Integer | Float | ~string
}

type Integer interface {
    Signed | Unsigned
}

type Signed interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type Unsigned interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr
}

type Float interface {
    ~float32 | ~float64
}

func Map2Splice[T any, K OrderedType](mapData map[K]T) []T {
    listData := make([]T, len(mapData))
    for _, t := range mapData {
        listData = append(listData, t)
    }
    return listData
}
```

# 一些方便的集合转换操作 

```go

func InArray[vT any](need vT, haystack []vT) bool {
	for _, v := range haystack {
		if cast.ToString(need) == cast.ToString(v) {
			return true
		}
	}
	return false
}

func ArrayMap[T, T2 any](f func(T) T2, list []T) (result []T2) {
	for _, item := range list {
		result = append(result, f(item))
	}
	return
}

func ArrayFilter[T any](f func(T) bool, list []T) (result []T) {
	for _, item := range list {
		if f(item) {
			result = append(result, item)
		}
	}
	return
}


func ArrayKeyExists(key any, arr map[any]any) bool {
	if len(arr) == 0 {
		return false
	}
	for k := range arr {
		if key == k {
			return true
		}
	}
	return false
}

func Map2Slice[K comparable, V any](m map[K]V) []V {
	s := make([]V, len(m))
	i := 0
	for _, v := range m {
		s[i] = v
		i += 1
	}
	return s
}


func Slice2Map[V any, K comparable](s []V, f func(V) K) map[K]V {
	m := make(map[K]V, len(s))
	for _, v := range s {
		m[f(v)] = v
	}
	return m
}

func Map2Map[mK comparable, mV any, newK comparable, newV any](oldMap map[mK]mV, f func(mK, mV) (newK, newV)) map[newK]newV {
	newMap := make(map[newK]newV, len(oldMap))
	for k, v := range oldMap {
		nk, nv := f(k, v)
		newMap[nk] = nv
	}
	return newMap
}

```
