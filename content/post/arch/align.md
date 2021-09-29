---
title: "浅谈Golang内存对齐"
date: 2021-09-29T10:42:31+08:00
draft: false
categories:
  - "实战"
---

如果你在 [golang spec](https://golang.org/ref/spec) 里以「alignment」为关键字搜索的话，那么会发现与此相关的内容并不多，只是在结尾介绍 unsafe 包的时候提了一下，不过别忘了字儿越少事儿越大：

Computer architectures may require memory addresses to be aligned; that is, for addresses of a variable to be a multiple of a factor, the variable’s type’s alignment. The function Alignof takes an expression denoting a variable of any type and returns the alignment of the (type of the) variable in bytes. For a variable x:

```golang
uintptr(unsafe.Pointer(&x)) % unsafe.Alignof(x) == 0
```

The following minimal alignment properties are guaranteed:

- For a variable x of any type: unsafe.Alignof(x) is at least 1.
- For a variable x of struct type: unsafe.Alignof(x) is the largest of all the values unsafe.Alignof(x.f) for each field f of x, but at least 1.
- For a variable x of array type: unsafe.Alignof(x) is the same as the alignment of a variable of the array’s element type.

<!--more-->

当然，如果你以前没有接触过内存对齐的话，那么对你来说上面的内容可能过于言简意赅，在继续学习之前我建议你阅读以下资料，有助于消化理解：

- [图解 Go 之内存对齐](http://blog.newbmiao.com/slides/%E5%9B%BE%E8%A7%A3Go%E4%B9%8B%E5%86%85%E5%AD%98%E5%AF%B9%E9%BD%90.pdf)
- [在 Go 中恰到好处的内存对齐](https://eddycjy.gitbook.io/golang/di-1-ke-za-tan/go-memory-align)
- [Go 结构体的内存布局](https://www.liwenzhou.com/posts/Go/struct_memory_layout/)
- [Golang 是否有必要内存对齐](https://ms2008.github.io/2019/08/01/golang-memory-alignment/)

## 测试

我构造了一个 struct，它有一个特征：字段按照一小一大的顺序排列，如果不看注释中的 Sizeof、Alignof、Offsetof 信息（通过 unsafe 获取），你能否说出它占用多少个字节？

```golang
package main

import (
	"fmt"
	"unsafe"
)

type memAlign struct {
	a byte     // Sizeof: 1  Alignof: 1 Offsetof: 0
	b int      // Sizeof: 8  Alignof: 8 Offsetof: 8
	c byte     // Sizeof: 1  Alignof: 1 Offsetof: 16
	d string   // Sizeof: 16 Alignof: 8 Offsetof: 24
	e byte     // Sizeof: 1  Alignof: 1 Offsetof: 40
	f []string // Sizeof: 24 Alignof: 8 Offsetof: 48
}

func main() {
	var m memAlign
	fmt.Println(unsafe.Sizeof(m))
}
```

初学者往往会认为 struct 的大小应该等于内部各个字段大小的和，于是得出本例的答案是 51（1+8+1+16+1+24=51），不过实际上答案却是 72！究其原因是因为内存对齐的缘故导致各个字段之间可能存在 padding。那么有没有简单的方法来减少 padding 呢？我们不妨把字段按照从大到小的顺序排列，再试一试：

```golang
package main

import (
	"fmt"
	"unsafe"
)

type memAlign struct {
	f []string // Sizeof: 24 Alignof: 8 Offsetof: 0
	d string   // Sizeof: 16 Alignof: 8 Offsetof: 24
	b int      // Sizeof: 8  Alignof: 8 Offsetof: 40
	a byte     // Sizeof: 1  Alignof: 1 Offsetof: 48
	c byte     // Sizeof: 1  Alignof: 1 Offsetof: 49
	e byte     // Sizeof: 1  Alignof: 1 Offsetof: 50
}

func main() {
	var m memAlign
	fmt.Println(unsafe.Sizeof(m))
}
```

结果答案变成了 56，比 72 小了很多，不过还是比 51 大，说明还是存在 padding，这是因为不仅字段要内存对齐，struct 本身也要内存对齐。

另：我刚学 golang 的时候一直有一个疑问：为什么切片的大小是 24，字符串的大小是 16 呢？我估计别的初学者也会有类似的问题，一并解释一下，这是因为切片和字符串也是 struct，其定义分别对应 [SliceHeader](https://pkg.go.dev/reflect#SliceHeader) 和 [StringHeader](https://pkg.go.dev/reflect#StringHeader)，它们的大小分别是 24 和 16：

```golang
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

type StringHeader struct {
	Data uintptr
	Len  int
}
```

因为 uintptr 的大小等于 int，所以切片的大小等于 3*8=24，字符串的大小等于 2*8=16。

## 工具

只要我们写点代码，调用 unsafe 包的 Sizeof、Alignof、Offsetof 等方法，那么就可以搞清楚 struct 内存对齐的各种细节，不过这毕竟是个没有技术含量的体力活，有没有相关工具可以提升我们的工作效率呢？答案是 [go-tools](https://github.com/dominikh/go-tools)：

```shell
shell> go install honnef.co/go/tools/cmd/structlayout@latest
shell> go install honnef.co/go/tools/cmd/structlayout-pretty@latest
shell> go install honnef.co/go/tools/cmd/structlayout-optimize@latest
```

其中，structlayout 是用来分析数据的，pretty 是用来图形化显示的，optimize 是用来优化建议的，这里就用文章开头优化前的代码给出一个 structlayout-pretty 的例子：

```shell
shell> structlayout -json ./main.go memAlign | structlayout-pretty
```

![structlayout-pretty](https://blog.huoding.com/wp-content/uploads/2021/09/structlayout-pretty.png)

虽然 structlayout-pretty 我们可以很直观的看到在哪里存在 padding，不过它是 ascii 风格的，有时候不太方便，此时另外一个图形化工具 [structlayout-svg](https://github.com/ajstarks/svgo/tree/master/structlayout-svg) 更爽：

```shell
shell> go install github.com/ajstarks/svgo/structlayout-svg@latest
```

把文章开头优化前后的代码分别用 structlayout-svg 生成结果：

```shell
shell> structlayout -json ./main.go memAlign | structlayout-svg
```

优化前：

![优化前](https://blog.huoding.com/wp-content/uploads/2021/09/before.png)

优化后：

![优化后](https://blog.huoding.com/wp-content/uploads/2021/09/after.png)

效果超赞是不是！不过如果我们要把工具集成到 CI 里，那么此类图形化工具就不合适了，好在我们的工具箱里还有宝贝，它就是 [fieldalignment](https://github.com/golang/tools/blob/master/gopls/doc/analyzers.md#fieldalignment)：

```shell
shell> go install golang.org/x/tools/...@latest
```

把文章开头优化前后的代码分别用 fieldalignment 生成结果：

```shell
shell> awk '$1 == "module" {print $2}' ./go.mod | xargs fieldalignment
```

优化前：struct of size 72 could be 56；优化后：struct with 32 pointer bytes could be 24。

如上可见，fieldalignment 准确判断出优化前代码的 struct size 存在优化空间；但是优化后代码的 pointer bytes 是什么鬼？按照文档中的说明，pointer bytes 的含义如下：

Pointer bytes is how many bytes of the object that the garbage collector has to potentially scan for pointers, for example:

```golang
struct { uint32; string }
```

have 16 pointer bytes because the garbage collector has to scan up through the string’s inner pointer.

```golang
struct { string; *uint32 }
```

has 24 pointer bytes because it has to scan further through the *uint32.

```golang
struct { string; uint32 }
```

has 8 because it can stop immediately after the string pointer.

看到这里，不禁让人产生疑惑：GC 不会这么傻吧，难道它还要一个字节一个字节的扫描内存么？让我们做个实验测试一下 pointer bytes 有没有影响，正所谓有病没病走两步：

```golang
package main

import (
	"runtime"
	"time"
)

// pointer bytes: 8
type foo struct {
	S string
	U uint32
}

// pointer bytes: 16
type bar struct {
	U uint32
	S string
}

// GODEBUG=gctrace=1 go run main.go
func main() {
	v := make([]foo, 1e8)
	// v := make([]bar, 1e8)
	for range time.Tick(time.Second) {
		runtime.GC()
	}
	runtime.KeepAlive(v)
}
```

代码里构造了一个巨大的切片变量，栈必然保存不了，于是变量会逃逸到堆，接着周期性的调用 runtime.GC 来手动触发 GC，然后执行的时候通过 GODEBUG=gctrace=1 获取实时的 GC 相关信息。结果显示，不管是小 pointer bytes 的 foo，还是大 pointer bytes 的 bar，最终 GC 消耗的时间差不多。换句话说，pointer bytes 的大小对 GC 的影响很小很小，在 golang 的相关 [issue](https://github.com/golang/go/issues/44877#issuecomment-794565908) 的讨论中，也能印证此结论，篇幅所限，这里就不多说了。

另：命令输出的 gctrace 信息比较多，相关格式说明可以参考 [runtime](https://pkg.go.dev/runtime) 中的注释信息。

## 例子

了解了内存对齐的相关知识后，让我们看看现实世界中的例子，首先是 [groupcache](https://github.com/golang/groupcache/blob/master/groupcache.go)：

```golang
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
```

通过注释我们可以看到，为了强制让 Stats 在 32 位平台上按 8 字节对齐，在 Stats 字段的前面加了一个「_ int32」，换句话说，就是加了 4 个字节，那么为什么要这么做？

原因是 Stats 字段要参与 atomic 原子运算，关于 [atomic](https://pkg.go.dev/sync/atomic)，文档最后记录了如下内容：

> On ARM, 386, and 32-bit MIPS, it is the caller’s responsibility to arrange for 64-bit alignment of 64-bit words accessed atomically. The first word in a variable or in an allocated struct, array, or slice can be relied upon to be 64-bit aligned.

也就是说，在 32 位平台，调用者有责任自己保证原子操作是 64 位对齐的，此外，struct 中第一个字段可以被认为是 64 位对齐的。在本例中，因为 Stats 字段要参与 atomic 运算，而且不是第一个字段，所以我们必须手动保证它是 64 位对齐的，不过加了 _ int32 就能保证是 64 位对齐的么？让我们写代码验证一下：

```golang
package main

import (
	"fmt"
	"unsafe"

	"github.com/golang/groupcache"
)

// GOARCH=386 go run main.go
func main() {
	var g groupcache.Group
	fmt.Println(unsafe.Offsetof(g.Stats))
}
```

结果显示在 32 位下运行，Stats 的 offset 是 176，是 8 的倍数，满足 64 位对齐。如果没有「_ int32」做 padding，那么 Stats 的 offset 将是 172，就不再是 8 的倍数了。

…

再看看 sync.WaitGroup 中内存对齐的例子：

```golang
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

首先，noCopy 是什么鬼，其实它的作用就像名字一样，它是如何实现的呢，看注释：

```golang
// noCopy may be embedded into structs which must not be copied
// after the first use.
//
// See https://golang.org/issues/8005#issuecomment-190753527
// for details.
type noCopy struct{}

// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```

实际上它只是起到标识的作用，以便 go vet 能够借此发现问题，详细说明在 [issue](https://github.com/golang/go/issues/8005#issuecomment-190753527) 中有描述，如果你在自己的项目里有类似 noCopy 的需求，那么也可以照猫画虎，

接下来是内存对齐相关的重头戏了，state1 字段是一个有 3 个元素的 uint32 数组，它会保存两种数据，分别是 statep 和 semap，其中，statep 要参与 atomic 运算，所以我们要保证它是 64 位对齐的。如果「uintptr(unsafe.Pointer(&wg.state1))%8 == 0」成立，那么取前两个 int32 做 statep，否则取后两个 int32 做 statep。

为什么可以这样做？因为「uintptr(unsafe.Pointer(&wg.state1))%8 == 0」成立的时候，前两个 int32 自然满足 64 位对齐；当「uintptr(unsafe.Pointer(&wg.state1))%8 == 0」不成立的时候， 其运算结果必然等于 4，此时我们正好可以把第一个 int32 当作是一个 4 字节的 padding，于是后两个字节的 int32 就又满足 64 位对齐了。

如果你认为自己理解了，那么思考一下，在定义 state1 的时候，如果不用 [3]int32，而是换成一个 int64 加上一个 int32，或者是一个 [12]byte，它们都是 12 个字节，是否可以？
