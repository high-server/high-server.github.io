---
title: "记一次对Makefile的重构"
date: 2021-08-19T15:57:16+08:00
draft: false

#Comment
gitment: true
---

如果你不了解 Makefile 的话，那么推荐看看阮一峰的文章「[Make 命令教程](https://www.ruanyifeng.com/blog/2015/02/make.html)」。本文通过一个重构的例子带你写出味道更好的 Makefile，让我们开始吧！

假设有一个名为 foo 的项目，用 golang 开发，在 docker 上部署，其 Makefile 如下：

```makefile
APP = $(shell basename ${CURDIR})
TAG = $(shell git log --pretty=format:"%cd.%h" --date=short -1)

.PHONY: build
build:
	go build -ldflags "-X 'main.version=${TAG}'" -o ./tmp/${APP} .

.PHONY: docker-config
docker-config: env
	TAG=${TAG} docker-compose config

.PHONY: docker-build
docker-build: env
	TAG=${TAG} docker-compose build

.PHONY: docker-push
docker-push: env
	TAG=${TAG} docker-compose push

.PHONY: docker-up
docker-up: env
	TAG=${TAG} docker-compose up

.PHONY: docker-down
docker-down:
	TAG=${TAG} docker-compose down
```

看上去很简洁，唯一需要说明的是在操作 docker-compose 的时候，传递了一个名为 TAG 的环境变量，表示项目当前所属的标签，看一下对应的 docker-compose.yml 文件：

```yml
version: "3.0"
services:
  server:
    image: docker.domain.com/foo:${TAG}
    build:
      context: .
      dockerfile: build/docker/Dockerfile
    ports:
      - "9090:9090"
      - "6060:6060"
```

此时出现了一个有待改进的地方：ports 信息重复，看一下对应的 config.toml 文件：

```ini
[rpc]
port = 9090

[debug]
port = 6060
```

其中，rpc 端口 9090，debug 端口 6060 最初是在 config.toml 文件里配置的，但是在 docker-compose.yml 文件又重复了一次，假设要修改的话，就需要修改多个地方。

此时我们很容易想到的解决方案是把端口信息也通过环境变量传递，就像 TAG 变量那样，为了获取端口信息，我还专门写了一个子命令 config：

```go
package cmd

import (
	"fmt"
	"os"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"
)

func NewConfigCmd() *cobra.Command {
	configCmd := &cobra.Command{
		Use: "config ",
		Run: config,
	}
	return configCmd
}

func config(cmd *cobra.Command, args []string) {
	if len(args) != 1 {
		_ = cmd.Usage()
		os.Exit(1)
	}
	key := args[0]
	value := viper.Get(key)
	fmt.Println(value)
}
```

确定了解决方案，让我们在看一下对应的 docker-compose.yml 文件：

```yml
version: "3.0"
services:
  server:
    image: docker.domain.com/${APP}:${TAG}
    build:
      context: .
      dockerfile: build/docker/Dockerfile
    ports:
      - "${RPC_PORT}:${RPC_PORT}"
      - "${DEBUG_PORT}:${DEBUG_PORT}"
```

此时没有硬编码的配置信息了，让我们再看看对应的 Makefile 文件：

```makefile
APP = $(shell basename ${CURDIR})
TAG = $(shell git log --pretty=format:"%cd.%h" --date=short -1)
RPC_PORT   = $(shell ./tmp/${APP} config rpc.port)
DEBUG_PORT = $(shell ./tmp/${APP} config debug.port)

.PHONY: build
build:
	go build -ldflags "-X 'main.version=${TAG}'" -o ./tmp/${APP} .

.PHONY: docker-config
docker-config: env
	APP=${APP} TAG=${TAG} RPC_PORT={RPC_PORT} DEBUG_PORT={DEBUG_PORT} docker-compose config

.PHONY: docker-build
docker-build: env
	APP=${APP} TAG=${TAG} RPC_PORT={RPC_PORT} DEBUG_PORT={DEBUG_PORT} docker-compose build

.PHONY: docker-push
docker-push: env
	APP=${APP} TAG=${TAG} RPC_PORT={RPC_PORT} DEBUG_PORT={DEBUG_PORT} docker-compose push

.PHONY: docker-up
docker-up: env
	APP=${APP} TAG=${TAG} RPC_PORT={RPC_PORT} DEBUG_PORT={DEBUG_PORT} docker-compose up

.PHONY: docker-down
docker-down:
	APP=${APP} TAG=${TAG} RPC_PORT={RPC_PORT} DEBUG_PORT={DEBUG_PORT} docker-compose down
```

不得不说，长长的环境变量实在是太丑了，好在 [docker-compose 支持 .env 文件](https://docs.docker.com/compose/environment-variables/)，于是我们可以把环境变量写入 .env 文件，然后让 docker-compose 命令从其中取数据：

```makefile
APP = $(shell basename ${CURDIR})
TAG = $(shell git log --pretty=format:"%cd.%h" --date=short -1)
RPC_PORT   = $(shell ./tmp/${APP} config rpc.port)
DEBUG_PORT = $(shell ./tmp/${APP} config debug.port)

.PHONY: env
env:
	echo "APP=${APP}" > .env; \
		echo "TAG=${TAG}" >> .env; \
		echo "RPC_PORT=${RPC_PORT}" >> .env; \
		echo "DEBUG_PORT=${DEBUG_PORT}" >> .env

.PHONY: build
build:
	go build -ldflags "-X 'main.version=${TAG}'" -o ./tmp/${APP} .

.PHONY: docker-config
docker-config: env
	docker-compose config

.PHONY: docker-build
docker-build: env
	docker-compose build

.PHONY: docker-push
docker-push: env
	docker-compose push

.PHONY: docker-up
docker-up: env
	docker-compose up

.PHONY: docker-down
docker-down:
	docker-compose down
```

在 Makefile 里，我们定义了一个 env 操作，并把它作为所有 docker-compose 操作的前置执行操作，终于不用再写长长的环境变量了，不过记得把 .env 写到 .gitignore 里！
