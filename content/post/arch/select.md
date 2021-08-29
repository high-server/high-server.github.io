---
title: "一个select死锁问题"
date: 2021-08-29T21:18:16+08:00
draft: false
categories:
  - "实战"

#Comment
gitment: true
---

话说前几天我遇到了一个死锁问题，当时想了一些办法糊弄过去了，不过并没有搞明白问题的细节，周末想起来便继续研究了一下，最终便有了这篇文章。

<!--more-->

让我们搞一段简单的代码来重现一下当时我遇到的问题：

```golang
package main

import "sync"

func main() {
	var wg sync.WaitGroup
	foo := make(chan int)
	bar := make(chan int)
	closing := make(chan struct{})
	wg.Add(1)
	go func() {
		defer wg.Done()
		select {
		case foo <- <-bar:
		case <-closing:
			println("closing")
		}
	}()
	// bar <- 123
	close(closing)
	wg.Wait()
}
```

运行后报错，提示死锁：

> fatal error: all goroutines are asleep – deadlock!

因为「foo <- <-bar」的写法不太常见，所以第一感觉是不是 select 的 case 语句只能操作一个 chan，不能同时操作多个 chan，于是我改了一下，每个 case 只读写一个 chan：

```golang
package main

import "sync"

func main() {
	var wg sync.WaitGroup
	foo := make(chan int)
	bar := make(chan int)
	closing := make(chan struct{})
	wg.Add(1)
	go func() {
		defer wg.Done()
		select {
		case v := <-bar:
			foo <- v
		case <-closing:
			println("closing")
		}
	}()
	close(closing)
	wg.Wait()
}
```

果然死锁消失了！似乎 select 中，每个 case 确实只能读写一个 chan。为了确认到底是不是这个原因，我又修改了一下最初有问题的代码，加上了「bar <- 123」，结果死锁也消失了。看来虽然我找到了解决问题的方法，但是并没有找到解释问题的原因。

周末在家躺在床上，想起我认识的一个 golang 大神总对我说的：一切问题的答案都在 [spec](https://golang.org/ref/spec) 里。于是挣扎着爬起来仔细翻阅关于 [select](https://golang.org/ref/spec#Select_statements) 的说明，终于发现了问题真正的原因：

> For all the cases in the statement, the channel operands of receive operations and the channel and right-hand-side expressions of send statements are evaluated exactly once, in source order, upon entering the “select” statement. The result is a set of channels to receive from or send to, and the corresponding values to send. Any side effects in that evaluation will occur irrespective of which (if any) communication operation is selected to proceed. Expressions on the left-hand side of a RecvStmt with a short variable declaration or assignment are not yet evaluated.

结合这段话，让我们再来看看 case 中的这行代码「foo <- <-bar」，当左手边（foo）是一个写 chan 的操作，右手边（<-bar）是一个读 chan 的操作的时候，会先执行右手边的操作，如果拿到结果后再选择 case 执行，如果拿不到结果就会一直堵塞，于是死锁。

如果你觉得自己已经完全明白了，那么不妨看看下面这段代码：

```golang
package main

import (
	"fmt"
	"time"
)

func talk(msg string, sleep int) <-chan string {
	ch := make(chan string)
	go func() {
		for i := 0; i < 5; i++ {
			ch <- fmt.Sprintf("%s %d", msg, i)
			time.Sleep(time.Duration(sleep) * time.Millisecond)
		}
	}()
	return ch
}

func fanIn(input1, input2 <-chan string) <-chan string {
	ch := make(chan string)
	go func() {
		for {
			select {
			case ch <- <-input1:
			case ch <- <-input2:
			}
		}
	}()
	return ch
}

func main() {
	ch := fanIn(talk("A", 10), talk("B", 1000))
	for i := 0; i < 10; i++ {
		fmt.Printf("%q\n", <-ch)
	}
}
```

当然会出现死锁，我的问题是为什么每次都是不多不少输出一半数据才死锁？请回答。
