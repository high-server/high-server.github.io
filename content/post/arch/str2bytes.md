---
title: "聊一个string和[]byte转换问题"
date: 2021-10-14T14:09:31+08:00
draft: false
categories:
  - "实战"
---

前几天闲聊的时候，[景埕](https://github.com/diogin)说网上很多 string 和 []byte 的转换都是有问题的，当时并没有在意，转过身没几天我偶然看到[字节跳动的一篇文章](https://mp.weixin.qq.com/s/Xoaoiotl7ZQoG2iXo9_DWg)，其中提到了他们是如何优化 string 和 []byte 转换的，我便问景埕有没有问题，讨论过程中学到了很多，于是便有了这篇总结。

让我们看看问题代码，类似的 string 和 []byte 转换代码在网上非常常见：

```golang
func StringToSliceByte(s string) []byte {
	l := len(s)
	return *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
		Data: (*(*reflect.StringHeader)(unsafe.Pointer(&s))).Data,
		Len:  l,
		Cap:  l,
	}))
}
```

大家之所以不愿意直接通过 []byte(string) 把 string 转换为 []byte，是因为那样会牵扯内存拷贝，而通过 [unsafe.Pointer](https://pkg.go.dev/unsafe#Pointer) 来做类型转换，没有内存拷贝，从而达到提升性能的目的。

问题代码到底有没有问题？其实当我把代码拷贝到 vscode 之后就有提示了：

> SliceHeader is the runtime representation of a slice. It cannot be used safely or portably and its representation may change in a later release. Moreover, the Data field is not sufficient to guarantee the data it references will not be garbage collected, so programs must keep a separate, correctly typed pointer to the underlying data.

首先，[reflect.SliceHeader](https://pkg.go.dev/reflect#SliceHeader) 作为 slice 的运行时表示，以后可能会改变，直接使用它存在风险；其次，Data 字段无法保证它指向的数据不被 GC 垃圾回收。

前一个问题还好说，但是后面提的 GC 问题则是个大问题！为什么会存在 GC 问题，我们不妨看看 reflect.SliceHeader 和 reflect.StringHeader 的定义：

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

如上所示，Data 的类型是 uintptr，虽然有一个 ptr 后缀，但是它本质上还是一个整型，并不是指针，也就是说，它并不会持有它指向的数据，所以数据可能会被 GC 回收。

知道了前因后果，那么让我们构造一段代码来证明存在 GC 问题：

```golang
package main

import (
	"fmt"
	"reflect"
	"runtime"
	"unsafe"
)

func main() {
	fmt.Printf("%s\n", test())
}

func test() []byte {
	defer runtime.GC()
	x := make([]byte, 5)
	x[0] = 'h'
	x[1] = 'e'
	x[2] = 'l'
	x[3] = 'l'
	x[4] = 'o'
	return StringToSliceByte1(string(x))
}

func StringToSliceByte1(s string) []byte {
	l := len(s)
	return *(*[]byte)(unsafe.Pointer(&reflect.SliceHeader{
		Data: (*(*reflect.StringHeader)(unsafe.Pointer(&s))).Data,
		Len:  l,
		Cap:  l,
	}))
}
```

注：因为静态字符串存储在 TEXT 区，不会被 GC 回收，所以使用了动态字符串。

当我们运行上面的代码，并不会输出 hello，而是会输出乱码，原因是对应的数据已经被 GC 回收了，如果我们去掉 runtime.GC() 再运行，那么输出大概率会恢复正常。

由此可见，因为 Data 是 uintptr 类型，所以任何对它的赋值都是不安全的。原本问题到这里就应该告一段落了，但是 unsafe.Pointer 文档里恰好就有一个直接对 Data 赋值的例子：Conversion of a reflect.SliceHeader or reflect.StringHeader Data field to or from Pointer.

```golang
var s string
hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
hdr.Data = uintptr(unsafe.Pointer(p))
hdr.Len = n
```

到底是文档有误，还是我们的推断错了，继续看文档里的说明：

> the reflect data structures SliceHeader and StringHeader declare the field Data as a uintptr to keep callers from changing the result to an arbitrary type without first importing “unsafe”. However, this means that SliceHeader and StringHeader are only valid when interpreting the content of an actual slice or string value.

也就是说，只有当操作实际存在的 slice 或 string 的时候，SliceHeader 或 StringHeader 才是有效的，回想最初的代码，因为操作 reflect.SliceHeader 的时候，并没有实际存在的 slice，所以是不符合 unsafe.Pointer 使用规范的，按照要求调整一下：

```golang
func StringToSliceByte(s string) []byte {
	var b []byte
	l := len(s)
	p := (*reflect.SliceHeader)(unsafe.Pointer(&b))
	p.Data = (*reflect.StringHeader)(unsafe.Pointer(&s)).Data
	p.Len = l
	p.Cap = l
	return b
}
```

再用测试代码跑一下，国藩发现输出正常了。不过有人可能会问了，之前不是说了 uintptr 不是指针，不能阻止数据被 GC 回收，可是为什么 GC 没有效果？实际上这是因为编译器对 *reflect.{Slice,String}Header 做了[特殊处理](https://github.com/golang/go/issues/19168)，具体细节不展开了。

如果你想验证是否存在特殊处理，可以使用自定义的类型反向验证一下：

```golang
type StringHeader struct {
	Data uintptr
	Len  int
}

type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

func StringToSliceByte(s string) []byte {
	var b []byte
	l := len(s)
	p := (*SliceHeader)(unsafe.Pointer(&b))
	p.Data = (*StringHeader)(unsafe.Pointer(&s)).Data
	p.Len = l
	p.Cap = l
	return b
}
```

你会发现，如果没有使用 reflect 里的类型，那么输出就又不正常了。从而反向验证了编译器确实对 *reflect.{Slice,String}Header 做了特殊处理。

现在，我们基本搞清楚了 string 和 []byte 转换中的各种坑，下面看看如何写出准确的转换代码，虽然编译器在其中耍了一些小动作，但是我们不应该依赖这些黑科技。

既然 uintptr 不是指针，那么我们改用 unsafe.Pointer，如此数据就不会被 GC 回收了：

```golang
type StringHeader struct {
	Data unsafe.Pointer
	Len  int
}

type SliceHeader struct {
	Data unsafe.Pointer
	Len  int
	Cap  int
}

func StringToSliceByte3(s string) []byte {
	var b []byte
	l := len(s)
	p := (*SliceHeader)(unsafe.Pointer(&b))
	p.Data = (*StringHeader)(unsafe.Pointer(&s)).Data
	p.Len = l
	p.Cap = l
	return b
}
```

不过更简单的做法是彻底抛弃 reflect 包，具体参考 [gin 中的 bytesconv](https://github.com/gin-gonic/gin/blob/master/internal/bytesconv/bytesconv.go)：

```golang
func StringToBytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(
		&struct {
			string
			Cap int
		}{s, len(s)},
	))
}

func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

至此，我们完美解决了 string 和 []byte 的转换问题。
