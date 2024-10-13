---
title: 'Go 如何实现堆'
date: 2024-06-07T10:52:00+08:00
# weight: 1
# aliases: ["/alias"]
tags: ["golang","堆"]
author: "sweetpear"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: ""
summary: "介绍如何使用container/heap包实现堆"
---
# container/heap
## 定义规则
小根堆（大根堆Less函数反向）
```go
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x any) {
	// Push and Pop use pointer receivers because they modify the slice's length,
	// not just its contents.
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() any {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}
```

## 使用
```go
func main() {
	h := &IntHeap{2, 1, 5}
	heap.Init(h)
	heap.Push(h, 3)
	fmt.Printf("minimum: %d\n", (*h)[0])
	for h.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(h))
	}
}
```

## 注意事项

1. 使用时要初始化堆
2. Push 或 Pop 时使用的是 heap 包提供的方法，而不是自己定义的方法
3. 定义中的 `func (h *IntHeap) Push(x any) ` 和 `func (h *IntHeap) Pop() any` **并不是** 对堆的真实操作，
只需要把元素追加到数组**尾部**或从数组**尾部**拿出一个元素。

### 原因
heap.Pop函数声明
```go
// Pop is equivalent to [Remove](h, 0).
func Pop(h Interface) any {
	n := h.Len() - 1
	h.Swap(0, n)
	down(h, 0, n)
	return h.Pop()
}
```

heap.Push函数声明
```go
func Push(h Interface, x any) {
	h.Push(x)
	up(h, h.Len()-1)
}
```

由源码可知，Pop操作首先将数组首尾元素互换，并且向下调整了堆的顺序，因此我们定义的Pop方法只需要拿到并移除数组最后一位元素即可。

Push同理，在执行我们定义的Push方法以后，heap.Push还会调用`up`调整堆顺序。

## 应用
topk问题  