---
title: "再谈Golang内存对齐"
date: 2021-09-30T10:42:31+08:00
draft: false
categories:
  - "实战"
---

关于 Golang 内存对齐，昨天已经写了一篇「[浅谈Golang内存对齐](https://blog.huoding.com/2021/09/29/951)」，可惜对一些细节问题的讨论语焉不详，于是便有了今天这篇「再谈Golang内存对齐」。

<!--more-->

让我们回想一下 [groupcache](https://github.com/golang/groupcache/blob/master/groupcache.go) 和 [sync.WaitGroup](https://github.com/golang/go/blob/master/src/sync/waitgroup.go) 中的做法，为了规避在 32 位环境下 atomic 操作 64 位数的 BUG，它们采取了截然不同的做法：

```golang
// groupcache
type Group struct {
	name string
	getter Getter
	peersOnce sync.Once
	peers PeerPicker
	cacheBytes int64
	mainCache cache
	hotCache cache
	loadGroup flightGroup
	_ int32 // force Stats to be 8-byte aligned on 32-bit platforms
	Stats Stats
}

// sync.WaitGroup
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	state1 [3]uint32
}

func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
```

**问题：为什么 groupcache 不用考虑外部地址，只要内部对齐就可以实现 64 位对齐？**

为了搞清楚这个问题，让我们回想一下 [atomic](https://pkg.go.dev/sync/atomic) 文档最后关于 64 位对齐的相关描述：

> On ARM, 386, and 32-bit MIPS, it is the caller’s responsibility to arrange for 64-bit alignment of 64-bit words accessed atomically. The first word in a variable or in an allocated struct, array, or slice can be relied upon to be 64-bit aligned.

其中我们关心的是最后一句话：变量、结构体、数组、切片的第一个字是 64 位对齐的。为了验证这一点，我构造了一个包含 int64 的 struct，看它的地址是否是 8 的倍数：

```golang
package main

import (
	"fmt"
	"time"
	"unsafe"
)

type foo struct {
	bar int64
}

// GOARCH=386 go run main.go
func main() {
	for range time.Tick(time.Second) {
		f := &foo{}
		p := uintptr(unsafe.Pointer(f))
		fmt.Printf("%p: %d, %d\n", f, p, p%8)
	}
}
```

按照常理来说，当我们在 32 位环境（GOARCH=386）下运行的时候，struct 的地址应该只能满足 32 位对齐，也就是 4 的倍数，不过测试发现，当 struct 里含有 int64 的时候，struct 的地址竟然满足 64 位对齐，也就是是 8 的倍数。既然外部已经是对齐的了，那么只要内部对齐就可以实现 64 位对齐。

**问题：为什么 sync.WaitGroup 不像 groupcache 那样实现 64 位对齐。**

两者之所以采用了不同的 64 位对齐实现方式，是因为两者的使用场景不同。在实际使用的时候，sync.WaitGroup 可能会被嵌入到别的 struct 中，因为不知道嵌入的具体位置，所以不可能通过预先加入 padding 的方式来实现 64 位对齐，只能在运行时动态计算。而 groupcache 则不会被嵌入到别的 struct 中，如果你硬要嵌入，可能会出问题：

```golang
package main

import (
	"github.com/golang/groupcache"
)

type foo struct {
	bar int32
	g groupcache.Group
}

// GOARCH=386 go run main.go
func main() {
	f := foo{}
	f.g.Stats.Gets.Add(1)
}
```

当我们在 32 位环境（GOARCH=386）下运行的时候，会看到如下 panic 信息：

> panic: unaligned 64-bit atomic operation

当我们在 32 位环境，按 4 字节对齐，所以 g 的偏移量是 4 而不是 8，如此一来，虽然 groupcache 内部通过 _ int32 实现了相对的 64 位对齐，但是因为外部没有实现 64 位对齐，所以在执行 atomic 操作的时候，还是会 panic（如果 bar 是 int64 就不会 panic）。

**问题：为什么 sync.WaitGroup 中的 state1 不换成 一个 int64 加一个 int32？**

我们知道 sync.WaitGroup 中的 state1 字段是一个有 3 个元素的 uint32 数组，它会保存两种数据，分别是 statep 和 semap，相当于一个是 int64，另一个是 int32。那为什么它不直接把一个 state1 字段替换成两个独立的字段，一个 int64 加一个 int32。

当然可以换，但是因为 sync.WaitGroup 可能会被嵌入到别的 struct 中，并且不知道嵌入的具体位置，所以还是需要在运行时动态计算是否要 padding，并且这个 padding 的工作要额外空间来承担，不能被独立的 int32 兼任。和原方案比无疑浪费了空间。

**问题：为什么 sync.WaitGroup 中的 state1 不换成 一个[12]byte？**

原方案中 state1 的类型是 [3]uint32，取两个 uint32 做 statep，剩下的一个 uint32 做 semap。为什么不换成 [12]byte，取 8 个 byte 做 statep，剩下的 4 个 byte 做 semap？

想要搞清楚这个问题，我们想要回顾一下 golang 关于内存对齐保证的描述：

- For a variable x of any type: unsafe.Alignof(x) is at least 1.
- For a variable x of struct type: unsafe.Alignof(x) is the largest of all the values unsafe.Alignof(x.f) for each field f of x, but at least 1.
- For a variable x of array type: unsafe.Alignof(x) is the same as the alignment of a variable of the array’s element type.

其中的重点是：对 struct 而言，它的对齐取决于其中所有字段对齐的最大值；对于 array 而言，它的对齐等于元素类型本身的对齐。因为 noCopy 的大小是 0，所以 struct 的对齐实际上就取决于 state1 字段的对齐。

- 当 state1 的类型是 [3]uint32 的时候，那么 struct 的对齐就是 4。
- 当 state1 的类型是 [12]byte 的时候，那么 struct 的对齐就是 1。

如果 state1 换成 [12]byte，那么因为 struct 的对齐是 1，会导致 struct 的地址不再是 4 的倍数，结果 uintptr(unsafe.Pointer(&wg.state1))%8 == 0 的判断也就没有意义了。
