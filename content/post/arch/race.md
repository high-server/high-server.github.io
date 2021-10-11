---
title: "从一个data race问题学到的"
date: 2021-10-11T13:09:31+08:00
draft: false
categories:
  - "实战"
---

前几天我在学习[内存屏障](https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C)的时候搜到一篇文章「[Golang Memory Model](https://fanlv.wiki/2020/06/09/golang-memory-model/)」，其中在介绍 CPU 缓存一致性的时候提到一个例子，带给我一些困惑，本文记录下解惑过程。

既然是在介绍 CPU 缓存一致性的时候举的例子，那么理所应当与此有关，看代码：

```golang
package main

import "time"

func main() {
	running := true
	go func() {
		println("start thread1")
		count := 1
		for running {
			count++
		}
		println("end thread1: count =", count)
	}()
	go func() {
		println("start thread2")
		for {
			running = false
		}
	}()
	time.Sleep(time.Hour)
}
```

当我们通过「go run main.go」运行代码的时候，会发现第一个 goroutine 永远不会结束，就好像 running = false 没有生效一样。对此，文章把原因归结为 CPU 缓存一致性中的线程可见性问题，可是我前后看了几遍也没有看出个所以然来。细心的小伙伴不难发现代码存在 data race 问题：多个 goroutine 并发读写 running 变量，不过当我们通过「go run -race main.go」再次运行代码的时候，有趣的事情发生了，第一个 goroutine 正常结束了！

理论上，既然存在 data race 问题，那么出现什么结果都可能，但是好奇心驱使我继续研究了一下，这次使用的工具是 [SSA](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/README.md)，它可以展现出从源代码到汇编的过程中，编译器都做了哪些工作，并且可以把结果生成 html 文件：

```shell
shell> GOSSAFUNC=main go build -gcflags="-N -l" ./main.go
```

SSA 工具最方便的地方是它可以把源代码和汇编通过颜色对应起来：

![main 函数的 ssa](https://files.catbox.moe/z2ffru.png)

说明：Golang 中的汇编一般指 Plan9 汇编，推荐阅读「[plan9 assembly 完全解析](https://github.com/cch123/golang-notes/blob/master/assembly.md)」。

不过为什么「running = false」这行源代码没有对应的汇编呢？这是因为 SSA 的工作单位是函数，上面的结果是 main 函数的结果，而「running = false」实际上属于 main 函数里第 2 个 goroutine，实际上就相当于 main.func2，重新运行 SSA：

```shell
shell> GOSSAFUNC=main.func2 go build -gcflags="-l -N" ./main.go
```

如此一来就能看到「running = false」这行源代码对应的汇编了：

![main.func2 函数的 ssa](https://files.catbox.moe/vps60h.png)

其中，PCDATA 是编译器插入的和 GC 相关的信息，在本例中可以忽略，剩下的几个 JMP 跳来跳去，好像是个圈哦，就是一个空 for，和「running = false」完全没有关系。

不过既然带有 race 检测的代码工作正常，那么不妨一并生成 SSA 看看结果如何：

```shell
shell> GOSSAFUNC=main.func2 go build -race -gcflags="-l -N" ./main.go
```

结果如下图所示，出了 JMP，还有 MOV 操作，正好对应「running = false」：

![main.func2 函数的 ssa](https://files.catbox.moe/eod3ct.png)

如此一来，我们的困惑终于解开了。问题代码中的循环之所以不会结束，和所谓的「CPU 缓存一致性中的线程可见性问题」并没有任何关系，只是因为编译器把部分代码看成死代码，直接优化掉了，这个过程称之为「[Dead code elimination](https://en.wikipedia.org/wiki/Dead_code_elimination)」，不过当激活 race 检测的时候，编译器并没有执行死代码的优化，所以程序看上去又正常了。

最后，推荐一篇文章，和本文的例子相似：[谈谈 Golang 中的 Data Race](https://ms2008.github.io/2019/05/12/golang-data-race/)（及[续](https://ms2008.github.io/2019/05/22/golang-data-race-cont/)）。
