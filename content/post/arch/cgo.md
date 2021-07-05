---
title: "实战CGO"
date: 2021-07-03T15:01:55+08:00
draft: false
#Comment
gitment: true
---

某项目要集成 PDF 文件的 OCR 功能，不过由于此功能技术难度太大，网络上找不到靠谱的开源实现，最终不得不选择 [ABBYY FineReader Engine](https://www.abbyy.com/ocr-sdk/) 的付费服务。可惜 ABBYY 只提供了 C++ 和 Java 两种编程语言的 SDK，而我们的项目采用的编程语言是 Golang，此时通常的集成方法是使用 C++ 或 Java 实现一个服务，然后在 Golang 项目里通过 RPC 调用服务，不过如此一来明显增加了系统的复杂度，好在 Golang 支持 CGO，让我们可以很方便的在 Golang 中使用 C 模块，本文总结了我在学习 CGO 过程中的心得体会。

<!--more-->

## Hello World

让我们看看一个 CGO 版本的 Hello, world 大概长什么样：

```golang
package main

/*
#include <stdio.h>

void say(const char *s) {
    puts(s);
}
*/
import "C"

func main() {
    hello()
}

func hello() {
    s := C.CString("Hello, World\n")
    C.say(s)
}
```

如上所示，通过「import “C”」来激活 CGO，并且所有 C 语言相关的代码都以注释的形式放在此行之上，中间不允许有空行，这样我们就可以在 Golang 代码里使用 C 模块了，看上去很简单，不过代码里存在内存泄漏，让我们修改一下代码，使问题更明显一点：

```golang
package main

/*
#include <stdio.h>

void say(const char *s) {
    puts(s);
}
*/
import "C"

func main() {
    for {
        hello()
    }
}

func hello() {
    s := C.CString("Hello, World\n")
    C.say(s)
}
```

运行程序后，我们可以单独开一个命令行窗口，通过运行 top 命令来监控进程的内存变化，会发现在循环调用 C 模块之后，进程的内存占用不断增加，究其原因，是因为通过 C.CString 创建的变量，会在 C 语言层面上分配内存，而在 Golang 语言层面上是不会负责管理相关内存的，所以我们需要通过 C.free 手动释放相关内存：

```golang
package main

/*
#include <stdio.h>
#include <stdlib.h>

void say(const char *s) {
    puts(s);
}
*/
import "C"
import "unsafe"

func main() {
    for {
        hello()
    }
}

func hello() {
    s := C.CString("Hello, World\n")
    defer C.free(unsafe.Pointer(s))
    C.say(s)
}
```

说明：代码中的 unsafe.Pointer 相当于 C 语言中的 void *。

## In Action

有些读者看到这里可能会有疑问：虽然 CGO 让我们可以在 Golang 里使用 C，但是文章开头提到的 ABBYY 并没有 C 的 SDK，只有 C++ 的 SDK，那么 CGO 支持 C++ 么？答案是否定的，不过我们可以通过 C 来适配 C++。

以 ABBYY 为例，假设它的安装目录是 /opt/ABBYY/FREngine12，并且通过 [ldconfig](https://linux.die.net/man/8/ldconfig) 把 /opt/ABBYY/FREngine12/Bin 目录加入到动态链接库的查找目录：

```shell
shell> echo "/opt/ABBYY/FREngine12/Bin" > /etc/ld.so.conf.d/abbyy.conf
shell> ldconfig
```

准备工作做好后使用 /opt/ABBYY/FREngine12/Samples/Hello 例子做代码范本：

先编写 OCR.cpp 文件的内容，不用在意技术细节，我放这些代码只是为了备份：

```cpp
#include <string>
#include "AbbyyException.h"
#include "BstrWrap.h"
#include "FREngineLoader.h"
#include "./OCR.h"

using namespace std;

void load() {
    LoadFREngine();
}

void unload() {
    UnloadFREngine();
}

void process(const char *inPath, const char *outPath) {
    string file = outPath;
    string extension = file.substr(file.find_last_of(".") + 1);
    FileExportFormatEnum format;

    if (extension == "pdf") {
        format = FEF_PDF;
    } else if (extension == "doc" || extension == "docx") {
        format = FEF_DOCX;
    } else if (extension == "ppt" || extension == "pptx") {
        format = FEF_PPTX;
    } else if (extension == "xls" || extension == "xlsx") {
        format = FEF_XLSX;
    } else {
        return;
    }

    const wchar_t *language = L"ChinesePRC,ChineseTaiwan,English";
    CSafePtr frDocument = 0;
    CSafePtr documentProcessingParams;
    CSafePtr pageProcessingParams;
    CSafePtr recognizerParams;

    try {
        CheckResult(FREngine->CreateFRDocumentFromImage(CBstr(inPath), 0, &frDocument));
        CheckResult(FREngine->CreateDocumentProcessingParams(&documentProcessingParams));
        CheckResult(documentProcessingParams->get_PageProcessingParams(&pageProcessingParams));
        CheckResult(pageProcessingParams->get_RecognizerParams(&recognizerParams));
        CheckResult(recognizerParams->SetPredefinedTextLanguage(CBstr(language)));
        CheckResult(frDocument->Process(documentProcessingParams));
        CheckResult(frDocument->Export(CBstr(outPath), format, 0));
    } catch (...) {
        return;
    }
}
```

再编写 OCR.h 文件的内容，要特别注意其中的「[extern “C”](https://stackoverflow.com/questions/1041866/what-is-the-effect-of-extern-c-in-c)」，有了它，当编译的时候，就会把 C++ 中的方法名链接成 C 的风格，如此一来，CGO 才能识别它：

```cpp
#ifdef __cplusplus
extern "C" {
#endif
void load();
void unload();
void process(const char *inPath, const char *outPath);
#ifdef __cplusplus
}
#endif
```

最后编写 OCR.go 文件的内容，因为 C/C++ 代码量比较大，所以在使用 CGO 的时候直接把 C/C++ 代码写在注释中就显得不合适了，此时更合适的方法是链接库：

```golang
package main

// #cgo CFLAGS: -I .
// #cgo LDFLAGS: -L . -L /opt/ABBYY/FREngine12/Bin/ -lFREngine -lOCR -lstdc++
// #include <stdlib.h>
// #include "OCR.h"
import "C"
import (
	"flag"
	"os"
	"unsafe"
)

func main() {
	flag.Parse()

	if flag.NArg() != 2 {
		os.Exit(1)
	}

	C.load()
	inPath := C.CString(flag.Arg(0))
	outPath := C.CString(flag.Arg(1))

	defer func() {
		C.unload()
		C.free(unsafe.Pointer(inPath))
		C.free(unsafe.Pointer(outPath))
	}()

	C.process(inPath, outPath)
}
```

假设目标文件都已经就绪，那么让我们分别看看如何构建静态链接库和动态链接库：

先看静态链接库，只要通过如下 ar 命令即可，在最终编译程序的时候，静态链接库会被编译到程序里，所以运行时不存在依赖问题，当然代价就是文件尺寸相对较大：

```shell
shell> ar -r libOCR.a *.o
```

再看动态链接库，只要通过如下 gcc 命令即可，和静态链接库相比，虽然它运行时存在依赖问题，但是它生成的文件尺寸相对较小，不过需要提醒的是，在之前编译目标文件的时候，需要在 CFLAGS 或 CXXFLAGS 参数中需要加入 -fpic 或者 -fPIC 选项，以便实现地址无关，至于 -fpic 和 -fPIC 的区别，可以参考 [Shared Libraries](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)：

```shell
shell> gcc -shared -o libOCR.so *.o
shell> cp libOCR.so /opt/ABBYY/FREngine12/Bin/
```

动态链接库还有一个有点是更新方便，如果多个程序依赖同一个动态链接库的时候，那么当动态链接库有问题的时候，直接更新它即可，相反如果多个程序依赖同一个静态链接库，那么当静态链接库有问题的时候，你不得不重新编译每一个程序。不过动态链接库的依赖关系本身很容易出问题，下图是我的 OCR 程序依赖关系，有点复杂啊：

![动态链接](/img/cgo/ld.png)

本文仅是 CGO 的入门笔记，想进一步了解的话，推荐阅读「[CGO 编程](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/readme.html)」，收摊儿。
