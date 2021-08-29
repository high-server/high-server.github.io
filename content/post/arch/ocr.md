---
title: "关于OCR项目的流水账"
date: 2021-08-16T16:16:16+08:00
draft: false
categories:
  - "实战"

#Comment
gitment: true
---

最近一直在开发某个 OCR 项目：底层用的是 ABBYY 提供的 FineReader 引擎，应用层把 FineReader 包装成 gRPC 对外提供服务，因为 FineReader 项目是 C++ 实现的，而我们团队使用的编程语言是 Golang，所以二者间通过 CGO 来完成交互。整个项目没有特殊的需求，只是鉴于 OCR 耗时较长，为了提升产品体验，要求在处理过程中：客户端可以主动退出；服务端能够实时返回已处理百分比。下面是根据需求画出来的流程图：

![流程图](https://blog.huoding.com/wp-content/uploads/2021/08/flow.png)

看上去很简单，不过我还是遇到不少问题，虽然这些问题主要都是一些细枝末节，基本上和 OCR 没什么关系，但是对别的项目还是会有所帮助的，下面让我一一道来。

<!--more-->

## 代码冗长

编程里一个常见的坏味道是代码冗长，比如最开始我的 main.go 就是如此，它足足有几百行代码之多，里面充斥着各种初始化配置，日志之类的操作。

为了规避此类问题，我引入了一个 initializer 的概念，用来统一初始化操作，比如 viper：

```golang
package initializer

import (
	"strings"

	"github.com/fsnotify/fsnotify"
	log "github.com/sirupsen/logrus"
	"github.com/spf13/viper"
)

func Viper(env string) error {
	if env == "" {
		env = "development"
	}
	viper.AutomaticEnv()
	viper.SetConfigName(env)
	viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
	viper.AddConfigPath(".")
	viper.AddConfigPath("./configs")
	viper.AddConfigPath("../configs")
	if err := viper.ReadInConfig(); err != nil {
		return err
	}
	viper.WatchConfig()
	viper.OnConfigChange(func(e fsnotify.Event) {
		log.Debugf("config file changed: %s", e.Name)
	})
	return nil
}
```

有了 initializer 之后，原本挤在一起的代码就可以分而治之，同时因为函数签名统一返回 error，所以可以统一进行错误处理，最终 main.go 代码行数大大降低：

```golang
var version string

func main() {
	var env string
	cobra.EnableCommandSorting = false
	cobra.OnInitialize(func() {
		check(initializer.Viper(env))
		check(initializer.Logrus())
		// ...
	})
	rootCmd := &cobra.Command{
		Use:     filepath.Base(os.Args[0]),
		Version: version,
	}
	rootCmd.PersistentFlags().StringVarP(
		&env, "env", "e", os.Getenv("SERVICE_ENV"), "env",
	)
	rootCmd.AddCommand(cmd.NewServerCmd())
	check(rootCmd.Execute())
}

func check(err error) {
	if err != nil {
		panic(err)
	}
}
```

除了 initializer 以外，其实我还引入了一个 provider 的概念，用来获取 sarama 等实例，也可以降低代码冗长的坏味道，提升复用性，篇幅所限，本文就不做赘述了。

## 同步异步

因为我之前一直在学习 Kafka，所以最初在架构选型的时候完全忽略了 gRPC 之类的同步架构，一门心思的想要以 Kafka 为中心打造一个基于事件的异步架构。此类极端的思想往往是个坏信号，实际上这就跟政治一样，不管是极左还是极右，通常都不可取。关于同步和异步，各取所长才是最合理的选择，判断方法：如果是业务逻辑的实现部分，那么倾向于选择使用同步；如果是业务逻辑完成之后的后续通知部分：强烈建议选择使用异步。具体请参考「[走出微服务误区：避免从单体到分布式单体](https://skyao.io/talk/202007-microservice-avoiding-distributed-monoliths/)」。

## Kafka 客户端

既然 Kafka 在架构中的地位如此重要，那么需要选择一下用哪个客户端，其 Golang 客户端主要有：[sarama](https://github.com/Shopify/sarama)、[confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)、[kafka-go](https://github.com/segmentio/kafka-go)，优缺点如下：

- sarama：它是最流行也是最难用的，文档很烂，API 封装太低级，暴露了过多 Kafka 协议的细节，而且还不支持 context 等新的 Golang 特色，实现上它把所有值都当指针传递，导致过多的动态内存分配，频繁的垃圾回收，大量的内存使用。
- confluent-kafka-go：它是基于 librdkafka 实现的 CGO，这意味着使用了这个包，你的代码就会依赖 C 库，和 sarama 相比，它的文档更好，但是同样不支持 context。
- kafka-go：前面关于 saram 和 confluent-kafka-go 的坏话都是它说的。

看上去似乎 kafka-go 最好，confluent-kafka-go 次之，sarama 最烂，可是当我问一个鹅厂小伙伴的时候，他说他们都用 sarama，信大厂得永生，于是乎我也决定选 sarama 了，事后证明这可能是一个糟糕的选择，sarama 虽然很流行，但是确实很难用。但是不管怎么说，使用 sarama 的案例相对更多，用起来也更安心些，不过用之前要清楚坑在哪：

- [Golang中如何正确的使用sarama包操作Kafka？](https://www.cnblogs.com/wishFreedom/p/15131600.html)
- [为什么不推荐使用Sarama Go客户端收发消息？](https://help.aliyun.com/document_detail/266782.html)

## Sarama 的版本

一开始用 sarama 的时候，就遭到了当头棒喝，遇到了如下错误：

> ERROR: Failed to open Kafka producer: kafka: client has run out of available brokers to talk to (Is your cluster reachable?)

反复确认才发现是版本问题，我们的服务端版本比较低（0.11.0.0），翻看 [sarama 的 changelog](https://github.com/Shopify/sarama/releases/tag/v1.27.1)，发现是在 1.27.1 开始切换到高版本的，如此说来只要使用 1.27.0 就可以了，同时务必记得把版本依赖写入 go.mod 文件中：

```
replace github.com/Shopify/sarama => github.com/Shopify/sarama v1.27.0
```

## 多个 goroutines 的协同

前面提到 sarama 有一个问题是暴露了过多 Kafka 协议的细节，这一点在使用 consumer 的时候可见一斑：因为 sarama 暴露了分区的细节，所以带来了很多麻烦，比如要关闭 consumer 的话，不得不先关闭每一个分区上的 PartitionConsumer，最后才可以关闭 consumer。不过话说回来，正好可以借机练习一下多个 goroutines 的协同：

```golang
type Watchman struct {
	waitGroup sync.WaitGroup
	consumer  sarama.Consumer
	closing   chan struct{}
}

func NewWatchmanFromConsumer(c sarama.Consumer) *Watchman {
	return &Watchman{
		consumer: c,
		closing:  make(chan struct{}),
	}
}

func (w *Watchman) Watch(topic string) (<-chan *sarama.ConsumerMessage, error) {
	msg := make(chan *sarama.ConsumerMessage)
	pids, err := w.consumer.Partitions(topic)
	if err != nil {
		return nil, err
	}
	for _, pid := range pids {
		pc, err := w.consumer.ConsumePartition(topic, pid, sarama.OffsetNewest)
		if err != nil {
			return nil, err
		}
		w.waitGroup.Add(1)
		go func() {
			defer w.waitGroup.Done()
			for {
				select {
				case msg <- <-pc.Messages():
				case <-w.closing:
					pc.Close()
					return
				}
			}
		}()
	}
	return msg, nil
}

func (w *Watchman) Close() {
	close(w.closing)
	w.waitGroup.Wait()
	w.consumer.Close()
}
```

说明：留意代码中是如何通过 waitGroup 和 closing 来处理多个 goroutines 的协同的。

## 编译错误

一般编译 Golang 代码不会遇到什么错误，但是因为我们的项目牵扯到 C++，所以在编译过程中还是遇到了一些莫名其妙的问题，下面逐一记录一下：

error adding symbols: DSO missing from command line：

在老版本的 binutils 里，ld 会自动递归地解析链接的 lib，不过从 2.22（ld -v）开始，ld 缺省激活了 –no-copy-dt-needed-entries 选项，如此一来，ld 不会再自动递归地解析链接的 lib，而是需要由用户来手动指定。知道了来龙去脉，不难想到如下解决方案：

- 手动：通过 -l 选项手动加载需要的库，比如需要 libz.so，就设置 -lz
- 自动：在 LDFLAGS 里添加 -Wl,–copy-dt-needed-entries 选项

推荐资料：[libpthread.so.0: error adding symbols: DSO missing from command line](https://stackoverflow.com/questions/19901934/libpthread-so-0-error-adding-symbols-dso-missing-from-command-line)

undefined reference to `__cxa_throw_bad_array_new_length’：

编译 libstdc++ 时，会使用命令 msgfmt。而 msgfmt 依赖 libstdc++.so.6，但编译时，gcc的编译系统会把 msgfmt 的依赖指向其自身的 libstdc++.so.6，而不是系统自带的libstdc++.so.6。如果 gcc 的版本比较老，就会导致 libstdc++.so.6 与 msgfmt 不兼容。

知道了来龙去脉，不难想到解决方案就是使用新版 gcc，更具体一点说是使用版本不低于 4.9 的 gcc（CentOS 7 上的 gcc 版本一般是 4.8.5），不过不推荐直接从源代码安装新版 gcc，其困难程度不是一般人能接受的，相对更可取的方法是通过 [scl](https://www.softwarecollections.org/en/) 安装 devtoolset：

```shell
shell> gcc -v
gcc version 4.8.5
shell> yum install centos-release-scl
shell> yum install devtoolset-7
shell> scl enable devtoolset-7 bash
shell> gcc -v
gcc version 7.3.1
shell> exit
shell> gcc -v
gcc version 4.8.5
```

关于 devtoolset 还有一个冷知识：devtoolset 和 gcc 的版本对应关系如下：

- devtoolset-3: gcc 4.9
- devtoolset-4: gcc 5
- devtoolset-6: gcc 6
- devtoolset-7: gcc 7
- devtoolset-8: gcc 8
-
你会发现没有版本 5，原因在 [Release Notes for Red Hat Developer Toolset 6.0](https://access.redhat.com/documentation/en-us/red_hat_developer_toolset/6/html-single/6.0_release_notes/index) 里说了：

> The version number of Red Hat Developer Toolset has been raised from 4.1 to 6.0 to align with the major version of GCC. There is no Red Hat Developer Toolset 5.

嗯，我承认这个无聊的问题困扰了我好几年，最终知道原因后感觉真是怅然若失啊。

## 条件编译

因为我们的服务底层是 FineReader 引擎，而且我们只有其 Linux 版本的 SDK，加上我们的本地开发环境是 MAC 系统，所以一开始我们在本地是没办法编译的，每次修改完代码我都会把代码传到 Linux 上编译，真是让人焦躁啊，好在 Golang 支持通过文件名来进行条件编译，比如我把原本的 abbyy.go 文件按操作系统拆分成 _linux.go 和 _darwin.go：

abbyy_linux.go：

```golang
package doc

// #cgo CFLAGS: -I .
// #cgo LDFLAGS: ${SRCDIR}/vendor/libabbyy.a -L /opt/ABBYY/FREngine12/Bin -lFREngine -lPortLayer -lstdc++
// #include <stdlib.h>
/*
void loadAbbyy();
int runAbbyy(const char *source, const char *destination, const char *status);
void unloadAbbyy();
*/
import "C"
import "unsafe"

func doJob(source, destination, status string) bool {
	csource := C.CString(source)
	cdestination := C.CString(destination)
	cstatus := C.CString(status)
	C.loadAbbyy()
	defer func() {
		C.unloadAbbyy()
		C.free(unsafe.Pointer(csource))
		C.free(unsafe.Pointer(cdestination))
		C.free(unsafe.Pointer(cstatus))
	}()
	return C.runAbbyy(csource, cdestination, cstatus) == 0
}
```

abbyy_darwin.go：

```golang
package doc

func doJob(source, destination, status string) bool {
	return false
}
```

拆分后，虽然我的 MAC 系统还是benign使用 FineReader 引擎，但是至少能够在本地开发环境正常编译了，处理一些非 CGO 类的问题绰绰有余了。

## 测试 gRPC

开发完成 gRPC 服务后，免不了要时不时的测试它，最开始我用的是 grpcurl，类似：

```shell
shell> grpcurl -plaintext -emit-defaults \
    -d '{"source":"/tmp/01.pdf","destination":"/tmp/02.pdf"}' \
    <address> Abbyy.OCR
```

不过命令行用起来总是不如 web 方便，于是借助 [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) 集成了 swagger：

```proto
syntax = "proto3";

package pkg.proto.v1;

import "google/api/annotations.proto";
import "protoc-gen-openapiv2/options/annotations.proto";

option (grpc.gateway.protoc_gen_openapiv2.options.openapiv2_swagger) = {
    info: {
        version: "1.0";
    };
};

service AbbyyService {
    rpc OCR(OCRRequest) returns (stream OCRResponse) {
        option (google.api.http) = {
            post: "/ocr"
            body: "*"
        };
    }
}

message OCRRequest {
    string source = 10;
    string destination = 20;
}

message OCRResponse {
    string action = 10;
    int32 percentage = 20;
}
```

通过 protoc 编译：

```shell
shell> protoc -I /path/to/proto \
    --go_out=. \
    --go_opt=paths=source_relative \
    --go-grpc_out=. \
    --go-grpc_opt=paths=source_relative \
    --grpc-gateway_out=. \
    --grpc-gateway_opt=paths=source_relative \
    --openapiv2_out=./api \
    /path/to/proto/*.proto
```

其中 protoc-gen-openapiv2 插件能够生成 swagger 所需的 json文件，最终效果如下：

[swagger](https://blog.huoding.com/wp-content/uploads/2021/08/swagger.png)

顺便说一句，为了部署方便，我用「//go:embed *」语法把整个 [swagger ui](https://swagger.io/tools/swagger-ui/) 打包进二进制文件了，不得不说，embed 真是爽啊，有兴趣的可以参考：[Go embed 简明教程](https://colobu.com/2021/01/17/go-embed-tutorial/)。

## 公共 proto

在编写 proto 的时候，我们用到了 [googleapis](https://github.com/googleapis/googleapis)，[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) 等项目里的公共 proto，这里牵扯到一个如何导入公共 proto 的问题，最常见的方法是把这些公共 proto 直接拷贝到项目目录中，但是如果有很多的项目需要用到这些公共 proto 的话，那么就不得不拷贝很多个副本，于是又有人把公共 proto 统一保存到独立的仓库中，然后其他项目在构建的时候都引用它，如此也不错，不过总觉得差点啥，最终我发现了完美的解决方案 [buf](https://buf.build/)：

先编写 buf.yaml 文件，主要用来声明依赖那些公共 proto：

```yaml
version: v1beta1
deps:
  - buf.build/beta/googleapis
  - buf.build/grpc-ecosystem/grpc-gateway
build:
  roots:
    - .
```

再编写 buf.gen.yaml 文件，主要用来声明使用哪些插件，如何生成需要的文件：

```yaml
version: v1beta1
plugins:
  - name: go
    out: .
    opt:
      - paths=source_relative
  - name: go-grpc
    out: .
    opt:
      - paths=source_relative
  - name: grpc-gateway
    out: .
    opt:
      - paths=source_relative
  - name: openapiv2
    out: ./api
```

准备好后，先用「buf mod update」命令生成 buf.lock 锁定版本信息，在用「buf generate」命令就可以生成我们要的各种 go 文件和 json 文件了：

```shell
shell> buf mod update
shell> buf generate
```

可见使用 buf 比直接使用 protoc 要方便很多，而且还有很多高级功能，相见 [buf 文档](https://docs.buf.build/)。

## 依赖工具

在使用 grpc-gateway 的时候，我们用到了其中的 protoc-gen-openapiv2 工具，实际上，grpc-gateway 有两个大版本，protoc-gen-openapiv2 在 v2 版本中，而在 v1 版本中对应的工具叫做 protoc-gen-swagger，可见明确依赖工具的版本非常重要。

目前[推荐](https://github.com/golang/go/wiki/Modules#how-can-i-track-tool-dependencies-for-a-module)的方法是在项目根目录创建名为 tools.go 的文件来记录依赖工具，比如：

```golang
package tools

import (
	// _ "github.com/cosmtrek/air"
	// _ "github.com/Shopify/sarama/tools/kafka-console-consumer"
	// _ "github.com/Shopify/sarama/tools/kafka-console-producer"
	_ "github.com/bufbuild/buf/cmd/buf"
	_ "github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway"
	_ "github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2"
	_ "google.golang.org/grpc/cmd/protoc-gen-go-grpc"
	_ "google.golang.org/protobuf/cmd/protoc-gen-go"
)
```

如此一来，当执行「go mod tidy」的时候，依赖工具的版本信息也会被 go.mod 记录下来，后续别人接手项目后，就很清楚的知道依赖什么工具，分别是什么版本了。
