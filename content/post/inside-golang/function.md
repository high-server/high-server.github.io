---
title: "Golang 函数和 C 函数深度对比"
date: 2021-05-26T17:25:49+08:00
draft: false
categories:
  - "Golang原理"

#Comment
gitment: true

# Theme-Defined params
#lead: "大家好，我们是搜狗商业技术团队！" # Lead text
#comments: false # Enable Disqus comments for specific page
authorbox: true # Enable authorbox for specific page
---

无论是什么语言，函数都是最常被使用到的东西。

我们对比一下 Golang 和 C 这两种语言的函数实现，进而我们能真正理解以下两个问题。  
- 为什么 C 语言只能有一个返回值，而 Golang 中可以返回多个？
- Golang 函数调用在性能上和 C 比有何差异？  

<!--more--> 

# 一、C 语言函数深究  

我们准备一段简单的函数调用代码。  

```c
#include <stdio.h>  
int func(int p){   
    return 1;
}  
int main()  
{  
    int i;  
    for(i=0; i<100000000; i++){  
        func(2);  
    }  
    return 0;  
}
```

用 gcc 来查看下汇编代码。

```sh
# gcc -S main.c
```

汇编源码如下：

```c
func:
        movl    %edi, -4(%rbp)
        movl    %esi, -8(%rbp)
        movl    %edx, -12(%rbp)
        movl    %ecx, -16(%rbp)
        movl    %r8d, -20(%rbp)
        movl    $1, %eax
        
main:
        movl    $5, %r8d
        movl    $4, %ecx
        movl    $3, %edx
        movl    $2, %esi
        movl    $1, %edi
        call    func
       
        movl    $0, %eax
```
可以看到，在C语言中： 

**主要通过寄存器传递参数**  
所以，C 语言函数的性能杠杠的。寄存器是整个计算机体系结构中访问最最快的存储了。只有当参数数量大于 6 的时候，才开始使用栈。

**固定 eax 寄存器返回数据**  
因为固定使用 eax 寄存器做返回数据之用，所以在 C 语言中无法支持多个返回值。我们接下来看看 Golang 是如何支持多个返回值的。

# 二、Golang 函数深究

同样先写一段最简单的函数调用代码。  

```golang
package main

func myFunction(p1, p2, p3,p4, p5 int) (int,int) {
	var a int = p1+p2+p3+p4+p5
	var b int = 3
	return a,b
}

func main() {
	myFunction(1, 2, 3, 4, 5)
}
```

然后查看其汇编代码。  

```sh
//为了方便查看，使用-N -l 参数，能阻止编译器对汇编代码的优化
#go tool compile -S -N -l main.go > main.s
```

结果是这样的：  
```c
"".main STEXT size=95 args=0x0 locals=0x38
		0x000f 00015 (main.go:7)		SUBQ	$56, SP		//在栈上分配56字节
		0x0013 00019 (main.go:7)		MOVQ	BP, 48(SP)	//保存BP
		0x0018 00024 (main.go:7)		LEAQ	48(SP), BP

        0x001d 00029 (main.go:8)        MOVQ    $1, (SP)	//第一个参数入栈
        0x0025 00037 (main.go:8)        MOVQ    $2, 8(SP)	//第二个参数入栈
        0x002e 00046 (main.go:8)        MOVQ    $3, 16(SP)	//第三个参数入栈
        0x0037 00055 (main.go:8)        MOVQ    $4, 24(SP)	//第四个参数入栈
        0x0040 00064 (main.go:8)        MOVQ    $5, 32(SP)	//第五个参数入栈
        0x0049 00073 (main.go:8)        CALL    "".myFunction(SB)

"".myFunction STEXT nosplit size=99 args=0x38 locals=0x18
        0x000e 00014 (main.go:3)	MOVQ	$0, "".~r5+72(SP)
        0x0017 00023 (main.go:3)	MOVQ	$0, "".~r6+80(SP)
        0x0020 00032 (main.go:4)	MOVQ	"".p1+32(SP), AX
        0x0025 00037 (main.go:4)	ADDQ	"".p2+40(SP), AX
        0x002a 00042 (main.go:4)	ADDQ	"".p3+48(SP), AX
        0x002f 00047 (main.go:4)	ADDQ	"".p4+56(SP), AX
        0x0034 00052 (main.go:4)	ADDQ	"".p5+64(SP), AX
        0x004b 00075 (main.go:6)	MOVQ	AX, "".~r5+72(SP)
        0x0054 00084 (main.go:6)	MOVQ	AX, "".~r6+80(SP)
```

可以看到，在Golang中：   

**使用栈来传递参数**  
栈是位于内存之中的，虽然有 CPU 中 L1、L2、L3的帮助，但平均每次访问性能仍然和寄存器没法比。所以 Golang 的函数调用开销肯定会比 C 语言要高。后面我们将用一个实验来进行量化的比较。

**使用栈来返回数据**  
不像 C 语言那样固定使用一个 eax 寄存器，Golang 是使用栈来返回值的。这就是为啥 Golang 可以返回多个值的根本原因。

# 最后，性能开销对比

我们的测试方法简单粗暴，直接调用空函数 1 亿次，再统计计算平均耗时。

**C函数编译运行测试：**  

```c
#include <stdio.h>  
int func(int p){   
    return 1;
}  
int main()  
{  
    int i;  
    for(i=0; i<100000000; i++){  
        func(2);  
    }  
    return 0;  
}
```

```sh
# gcc main.c -o main
# time ./main
```  

第一次执行耗时大约是 0.339 s。

但这个耗时中包含了两块。一块是函数调用开销，另外一块是 for 循环的开销(其它的代码调用因为只有 1 次，而函数调用和 for 循环都有 1 亿次，所以直接就可以忽略了)。

所以我们得减去 for 循环的开销。接着我手工注释掉对函数的调用，只是空循环 100000000 次。

```c
int main()  
{  
    int i;  
    for(i=0; i<100000000; i++){  
    //    func(2);  
    }  
    return 0;  
}

```
这次总耗时是 0.314 s。

这样就计算出平均每次函数调用耗时 = (0.339s - 0.314s) / 100000000 = 0.25ns

**Golang函数编译运行**

```golang
func hello(a int) int {
	return 2
}

func main(){
	for i:=0; i<100000000; i++ {
		hello(1)
	}
}
```

```sh
# go build -gcflags="-m -l" main.go
```  

同样采用上述方法测出平均每次函数调用耗时 = (0.302s - 0.056 s) / 100000000 = 2.46ns   
  
可见 Golang 的函数调用性能还是比 C 要差一些。但再给大家个参考一下 PHP 的数据，之前我测过 PHP7 每次函数调用开销大约在 50 ns 左右。所以 Golang 虽然比不上 C，但总的来说性能还是不错的。
