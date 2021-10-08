---
title: "为什么会有atomic.LoadInt32"
date: 2021-10-08T17:09:31+08:00
draft: false
categories:
  - "实战"
---

前些天我们聊了 [Golang 内存对齐](https://blog.huoding.com/2021/09/29/951)的话题，后来我突然想到另一个问题：为什么会有 [atomic.LoadInt32](https://pkg.go.dev/github.com/hslam/atomic#LoadInt32)？可能你觉得思维太跳跃了，容我慢慢道来：首先，有 [atomic.LoadInt64](https://pkg.go.dev/github.com/hslam/atomic#LoadInt64) 很正常，因为对一个 int64 来说，它的大小是 8 个字节，如果是 32 位平台的话（字长 4 字节），CPU 一次最多操作 4 个字节，需要两次才能拿到全部数据，所以封装一个 atomic.LoadInt64 来实现原子操作；但是，对一个 int32 数据来说，它的大小是 4 字节，不管是 32 位平台（字长 4 字节），还是 64 位平台（字长 8 字节），CPU 应该都可以保证一次操作拿到数据，换句话说，如果读取一个 int32 数据，那么本身就应该是原子的，可是为什么会有 atomic.LoadInt32，这不是脱了裤子放屁么？

<!--more-->

有病没病走两步，让我们写一段代码来验证一下：

```golang
package main

import "sync/atomic"

var v = int32(0)

func main() {
	var x int32
	x = v // main.go:9
	_ = x
	x = atomic.LoadInt32(&v) // main.go:11
	_ = x
}
```

通过「go tool compile」运行代码，拿到对应的汇编结果：

```shell
shell> go tool compile -N -l -S main.go

0x0016 00022 (main.go:9)        MOVL    "".v(SB), AX
0x001c 00028 (main.go:9)        MOVL    AX, "".t+4(SP)
0x0020 00032 (main.go:11)       MOVL    "".v(SB), AX
0x0026 00038 (main.go:11)       MOVL    AX, "".t+4(SP)
```

不管是「x = v」还是「x = atomic.LoadInt32(&v)」，对应的汇编结果一摸一样。问题越来越有趣了，让我们看看是否能从 [sync/atomic](https://github.com/golang/go/tree/master/src/sync/atomic) 的源代码中找到答案：

Golang 代码中只有函数声明，实际上是使用汇编实现的：

```
// doc.go
func LoadInt32(addr *int32) (val int32)

// asm.s
TEXT ·LoadInt32(SB),NOSPLIT,$0
	JMP runtime∕internal∕atomic·Load(SB)
```

顺着路径，跳转到 [runtime/internal/atomic](https://github.com/golang/go/tree/master/src/runtime/internal/atomic)，会发现每个平台都有独立的 Load 实现：

在 amd64 平台，Load 是用 Golang 实现的，等价于直接读取：

```golang
func Load(ptr *uint32) uint32 {
	return *ptr
}
```

在 arm64 平台，Load 是用汇编实现的，并不是简单的一次操作：

```
TEXT ·Load(SB),NOSPLIT,$0-12
	MOVD	ptr+0(FP), R0
	LDARW	(R0), R0
	MOVW	R0, ret+8(FP)
	RET
```

结果跃然纸上：atomic.LoadInt32 之所以存在，是因为某些平台存在特殊性，所以我们需要封装一个统一的操作，如此更有利于我们写出平台无关的代码。
