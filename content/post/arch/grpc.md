---
title: "浅谈K8S下gRPC负载均衡问题"
date: 2021-07-14T18:17:05+08:00
draft: false
#Comment
gitment: true
---

一般来说，在 K8S 下部署服务是很简单的事儿，但是如果部署的是一个 gRPC 服务的话，那么稍不留神就可能掉坑里，个中缘由，且听我慢慢道来。

<!--more-->

在 K8S 下部署服务，缺省情况下会被分配一个地址（也就是 [ClusterIP](https://kubernetes.io/docs/concepts/services-networking/service/)），客户端的请求会发送给它，然后再通过负载均衡转发给后端某个 pod：

![ClusterIP](/img/grpc/cluster_ip.png)

如果是 HTTP/1.1 之类的服务，那么 ClusterIP 完全没有问题；但是如果是 gRPC 服务，那么 ClusterIP 会导致负载失衡，究其原因，是因为 gRPC 是基于 HTTP/2 的，多个请求在一个 TCP 连接上多路复用，一旦 ClusterIP 和某个 pod 建立了 gRPC 连接后，因为多路复用的缘故，所以后续其它请求也都会被转发给此 pod，结果其它 pod 则完全被忽略了。

看到这里，有的读者可能会有疑问：HTTP/1.1 不是实现了基于 KeepAlive 的连接复用么？为什么 HTTP/1.1 的复用没问题，而 HTTP/2 的复用就有问题？答案是 HTTP/1.1 的 复用是串行的，当请求到达的时候，如果没有空闲连接那么就新创建一个连接，如果有空闲连接那么就可以复用，同一个时间点，连接里最多只能承载有一个请求，结果是 HTTP/1.1 可以连接多个 pod；而 HTTP/2 的复用是并行的，当请求到达的时候，如果没有连接那么就创建连接，如果有连接，那么不管其是否空闲都可以复用，同一个时间点，连接里可以承载多个请求，结果是 HTTP/2 仅仅连接了一个 pod。

了解了 K8S 下 gRPC 负载均衡问题的来龙去脉，我们不难得出如下解决方案：

在 Proxy 中实现负载均衡：采用 Envoy 做代理，和每台后端服务器保持长连接，当客户端请求到达时，代理服务器依照规则转发请求给后端服务器，从而实现负载均衡。

![Proxy](/img/grpc/proxy.png)

在 Client 中实现负载均衡：把服务部署成 [headless service](https://kubernetes.io/docs/concepts/services-networking/service)，这样服务就有了一个域名，然后客户端通过域名访问 gRPC 服务，DNS resolver 会通过 DNS 查询后端多个服务器地址，然后通过算法来实现负载均衡。

![Client](/img/grpc/client.png)

两种方案的优缺点都很明显：Proxy 方案结构清晰，客户端不需要了解后端服务器，对架构没有侵入性，但是性能会因为存在转发而打折扣；Client 方案结构复杂，客户端需要了解后端服务器，对建构有侵入性，但是性能上更有优势。
