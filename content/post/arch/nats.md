---
title: "浅谈NATS消息系统"
date: 2021-05-24T10:28:44+08:00
draft: false
categories:
  - "实战"

#Comment
gitment: true
---

我用过很多消息系统，比如：简单的 [Redis Streams](https://redis.io/topics/streams-intro)；高效的 [Kafaka](https://kafka.apache.org/) 等等，不过自从我把编程语言切换到 Golang 以后，总觉得必须找个用 Golang 开发的消息系统才配得上门当户对，原本我已经和小家碧玉的 [NSQ](https://nsq.io/) 厮守终生，不过当我认识了上流社会 [CNCF](https://landscape.cncf.io/) 钦定的大家闺秀 [NATS](https://nats.io/) 后，刹那间就仿佛徐志摩遇到了林徽因，扭头就给结发妻子写了休书。

<!--more-->

## INSTALLATION

服务端 [nats-server](https://github.com/nats-io/nats-server)，客户端 [nats](https://github.com/nats-io/natscli)，监控工具 [nats-top](https://github.com/nats-io/nats-top)，性能测试工具 [nats-bench](https://github.com/nats-io/nats.go/tree/master/examples/nats-bench)：

```shell
shell> go get github.com/nats-io/nats-server/v2

shell> git clone https://github.com/nats-io/natscli.git
shell> cd natscli/nats
shell> go get .

shell> go get github.com/nats-io/nats-top

shell> git clone https://github.com/nats-io/nats.go.git
shell> cd nats.go/examples/nats-bench
shell> go get .
```

需要说明的是，关于 stream 有新旧两种架构的服务端实现，其中旧的 NATS Streaming Server 架构已经过时，如果你是初学者，直接使用新的 NATS JetStream 架构即可。

## BENCH

开多个命令行窗口，分别启动 nats-server，nats-top，nats-bench：

```shell
shell> nats-server -js -m 8222
shell> nats-top
shell> nats-bench -n 100000000 -np 10 -ms 1 a
```

![nats-top](/img/nats/nats-top.png)

如上所示，高达一千万的 MPS，我就问你 OK 不 OK！Beautiful 不 Beautiful！

## MODE

### [PUBLISH SUBSCRIBE](https://docs.nats.io/nats-concepts/pubsub)：

NATS 实现了一对多发布订阅消息模型。当 publisher 往 subject 上发布一条消息后，此 subject 上所有 subscriber 都能收到 此消息，属于一种广播。

![Publish Subscribe](/img/nats/publish_subscribe.png)

```shell
shell> nats sub source.subject
Subscribing on source.subject
[#1] Received on "source.subject"
ZNQz8dCWc5
[#2] Received on "source.subject"
d1EggZJYVT

shell> nats sub source.subject
Subscribing on source.subject
[#1] Received on "source.subject"
ZNQz8dCWc5
[#2] Received on "source.subject"
d1EggZJYVT

shell> nats pub source.subject "{{Random 10 10}}" --count 2
Published 10 bytes to "source.subject"
Published 10 bytes to "source.subject"
```

### [QUEUE GROUPS](https://docs.nats.io/nats-concepts/queue)：

如果我们把 subscriber 分组，那么当 publisher 往 subject 上发布一条消息后，同一组里只有一个 subscriber 会收到此消息，从而实现了负载均衡。

![Queue Groups](/img/nats/queue_groups.png)

```shell
shell> nats sub source.subject --queue foo
Subscribing on source.subject
[#1] Received on "source.subject"
LFuJZBjnxV

shell> nats sub source.subject --queue foo
Subscribing on source.subject
[#1] Received on "source.subject"
76kAIoUYCI

shell> nats pub source.subject "{{Random 10 10}}" --count 2
Published 10 bytes to "source.subject"
Published 10 bytes to "source.subject"
```

### [REQUEST REPLY](https://docs.nats.io/nats-concepts/reqreply)：

一般来说，消息系统是以异步的形式工作，也就是说，publisher 往 subject 上发布一条消息后，并不在意 subscriber 的 reply 是什么。如果 publisher 在意 subscriber 的 reply 是什么的话，那么消息系统就应该以同步的形式工作，在具体实现中，是通过两次发布订阅来完成的：当 publisher 发布消息后，它会订阅一个特定的 subject，当 subscriber 处理完消息后，它会把 reply 发布到这个特定的 subject。当然，整个过程对使用者是透明的。

![Request Reply](/img/nats/request_reply.png)

```shell
shell> nats reply 'weather.>' --command "curl -s wttr.in/{{1}}?format=1"
Listening on "weather.>" in group "NATS-RPLY-22"
[#0] Received on subject "weather.beijing":

shell> nats request weather.beijing ''
Sending request on "weather.beijing"
Received on "_INBOX.7mc3ox00ma7WYWyNjuBSsw.NBtCmYbp"
☀️ +30°C
```

通过 weather 例子，我们可以发现 request reply 模式已经有了 RPC 的味道。

## MICROSERVICE

正是因为 NATS 具备了 RPC 的能力，所以在微服务中采用 NATS 后，系统会更清晰。

![传统微服务架构](/img/nats/microservice1.png)

![采用 NATS 的微服务架构](/img/nats/microservice2.png)

说明：以上图片来自于「[The Zen of High Performance Messaging with NATS](https://www.slideshare.net/nats_io/the-zen-of-high-performance-messaging-with-nats-76985268)」。

## MONITOR

说到监控，除了前面提到的 nats-top 之外，还有诸如 [natsboard](https://github.com/devfacet/natsboard) 之类的 UI 可供选择：

![natsboard](/img/nats/natsboard.png)

现实中，大家都知道，徐志摩和林徽因的结局，终究还是错付了，不过我对 NATS 的爱不会变，她是我的不二之选，至少在更好的消息系统出现前如此。
