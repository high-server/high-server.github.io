---
title: "记又一次对Makefile的重构"
date: 2021-08-21T14:42:19+08:00
draft: false
categories:
  - "实战"

#Comment
gitment: true
---

我平常有一个习惯，就是不断看以前写的代码，想着有没有哪些方面可以改进，如果每天能把代码可读性量变​ 1%，那么日积月累就是质变：前些天我们写过一次对 Makefile 的重构，去掉了一处重复代码的坏味道，没过多久我便又发现了一处重复代码的坏味道，本文就让我们看看如何消灭它！

<!--more-->

让我们先把问题的来龙去脉搞清楚，在 Golang 项目里，一般[推荐](https://github.com/golang/go/wiki/Modules#how-can-i-track-tool-dependencies-for-a-module)在根目录创建一个名为 tools.go 的文件，里面记录本项目依赖的相关工具，比如我的某个项目的 tools.go 如下：

```go
// +build tools

package tools

import (
	// _ "github.com/cosmtrek/air"
	// _ "github.com/goreleaser/goreleaser"
	_ "github.com/bufbuild/buf/cmd/buf"
	_ "github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway"
	_ "github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2"
	_ "github.com/tomwright/dasel/cmd/dasel"
	_ "google.golang.org/grpc/cmd/protoc-gen-go-grpc"
	_ "google.golang.org/protobuf/cmd/protoc-gen-go"
)
```

如此一来，当执行「go mod tidy」的时候，依赖工具的版本信息就会记录到 go.mod，接下来一般推荐在 Makefile 里创建一个 dep 操作，用来安装（make dep）依赖工具：

```makefile
.PHONY: dep
dep:
	@go install \
		github.com/bufbuild/buf/cmd/buf \
		github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway \
		github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2 \
		github.com/tomwright/dasel/cmd/dasel \
		google.golang.org/grpc/cmd/protoc-gen-go-grpc \
		google.golang.org/protobuf/cmd/protoc-gen-go
```

看上去不错，但是细心的你估计已经发现重复代码的坏味道了：tools.go 和 Makefile 文件内容重复了，以后如果想要增加一个依赖工具的话，那么两个文件都要改！

下面让我们看看如何重构：tools.go 和 Makefile 比起来，肯定 tools.go 更重要，它是不能改的，所以我们要去掉 Makefile 里的重复代码，更具体点来说是最好能在 Makefile 里通过 解析 tools.go 来确定想要执行的 go install 操作，这不就是 awk 擅长的工作么：

```makefile
.PHONY: dep
dep:
	@awk '$1 == "_" { print $2 | "go install" }' ./tools.go
```

看，通过一行 awk 代码，我们神奇的去掉了原本一坨重复代码，完美！
