---
title: "在docker环境导入私有仓库的问题"
date: 2021-08-24T15:55:52+08:00
draft: false
categories:
  - "实战"

#Comment
gitment: true
---

最近我遇到了一个在 docker 环境导入私有仓库的问题：一个 Golang 项目，使用 [gitlab ci](https://docs.gitlab.com/ee/ci/) 来发布，通过 [gitlab runner](https://docs.gitlab.com/runner/) 调用 [docker-compose](https://docs.docker.com/compose/) 来打包，但是在构建时失败了。

让我们重回案发现场，看看是否留下了什么线索：

首先是 .gitlab-ci.yml 文件，其相关代码片段内容如下：

```yml
build_job:
  stage: build
  script:
    - make docker-build
```

然后是 Makefile 文件，其相关代码片段内容如下：

```yml
.PHONY: docker-build
docker-build:
	@docker-compose build
```

接着是 docker-compose.yml 文件，其相关代码片段内容如下：

```yml
build:
  context: .
  dockerfile: Dockerfile
```

最后是 Dockfile 文件，其相关代码片段内容一下：

```
FROM golang:1.17 AS builder
WORKDIR /go/src/app
COPY . .
RUN go build
```

结果在 build 的时候报错了：

> fatal: could not read Username for ‘https://git.domain.com’: terminal prompts disabled

因为 git.domain.com 是一个私有仓库，所以问题乍一看上去会以为是 GOPRIVATE 和 GOPROXY 的配置有问题，不过我的配置都是 OK 的：

```shell
shell> go env -w GOPRIVATE=git.domain.com
shell> go env -w GOPROXY=https://goproxy.cn,direct
```

实际上，根本原因是因为访问私有仓库的时候是需要用户名和密码的，但是在 docker 容器里获取不到用户名密码，所以就报错了。下面看看我是如何解决问题的：

## 第一次尝试
既然问题出在用户名密码上，那么把仓库改成公开的不就可以了么？可惜结果报错：

> Visibility level public is not allowed in a private group.

我用的是 gitlab，它不允许在私有组里搞一个公开项目。

## 第二次尝试
既然搞不成公开项目，那么就想办法传递用户名密码吧，不过我们在使用 git 的时候，一般不会直接使用用户名密码，而是使用 KEY 来访问仓库，下面举例说明一下如何传递私钥参数 SSH_PRIVATE_KEY（其中牵扯到一个 docker 构建参数的概念）：

首先因为此类信息比较敏感，所以应该避免硬编码，我们选择在 gitlab 里创建它：

![Secret variables: settings > Pipelines](https://blog.huoding.com/wp-content/uploads/2021/08/variable.png)

接着是 docker-compose.yml 文件，其相关代码片段内容如下：

```yml
build:
  context: .
  dockerfile: Dockerfile
  args:
    - SSH_PRIVATE_KEY
```

最后是 Dockfile 文件，其相关代码片段内容一下：

```
FROM golang:1.17 AS builder
ARG SSH_PRIVATE_KEY
WORKDIR /go/src/app
COPY . .
RUN umask 0077 \
    && mkdir -p ~/.ssh \
    && echo "${SSH_PRIVATE_KEY}" > ~/.ssh/id_rsa \
    && ssh-keyscan git.domain.com >> ~/.ssh/known_hosts \
    && git config --global url."git@git.domain.com:".insteadOf https://git.domain.com/
RUN go build
```

此方法可以解决问题，但是把敏感信息传来传去总觉得不安心，容易出问题，推荐：[Access Private Repositories from Your Dockerfile Without Leaving Behind Your SSH Keys](https://vsupalov.com/build-docker-image-clone-private-repo-ssh-key/)。

## 第三次尝试

如果不想把敏感信息传来传去，那么还有没有安全的解决方案呢？答案是肯定的！我们只要在 gitlab runner 里执行「go mod vendor」就可以了，这是因为 gitlab runner 已经缓存了 git 认证信息，它可以访问所有的私有仓库，当执行「go mod vendor」后，项目依赖就都被放到 vendor 目录里了，接下来当执行到 Dockerfile 的 COPY 指令时，依赖就被自然而然的拷贝到了容器中，从而不用再联网执行 git 下载。

下面是修改后的 .gitlab-ci.yaml 文件，其相关代码片段内容如下：

```yml
build_job:
  stage: build
  script:
    - go mod vendor
    - make docker-build
```

也就是说，我们只加了一行代码「go mod vendor」，就解决了问题，是不是很简洁。最后友情提示一下：记得把 vendor 目录放到 .gitignore 里哦。
