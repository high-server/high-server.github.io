---
title: "浅谈微服务"
date: 2021-05-18T10:17:01+08:00
draft: false
categories:
  - "架构技术"
  
#Comment
gitment: true
---

虽说[微服务](https://martinfowler.com/articles/microservices.html)是一个老生常谈的话题，在 [infoq](https://www.infoq.cn/topic/microservice) 或者 [thoughtworks](https://insights.thoughtworks.cn/tag/microservices/) 上可以找到很多案例，不过可惜的是其中相当比例的案例是失败的案例，究其原因，除了[技术门槛](https://microservices.io/index.html)之外，主要是因为很多人脱离了实际情况，只是为了微服务而微服务。本文通过一个例子带领大家从头到尾体验一下微服务的演化过程，不仅要做到知其然，更要做到知其所以然。

<!--more-->

假设我们正在开发一个在线购物项目，其主要功能包括商城、推荐、评论、用户等，它是一个典型的[单体架构](https://microservices.io/patterns/monolithic.html)：不同团队的技术人员工作在同一个版本库上，系统功能按模块划分，不同模块之间通过本地函数调用，通常操作同一个数据库。

![微服务](/img/microservice/01.png)

在项目早期，单体架构往往能很好的适应快速迭代的需求，不过随着项目的发展，项目本身会变得复杂，其弊端不可避免的出现，比如下面列举的一些情况：

- 因为大家都工作在同一个版本库上，所以可能会遇到：商城模块完成了新功能，准备上线，结果推荐模块刚提交了还没来得及测试的代码，于是不得不推迟上线。
- 不同的需求采用不同的技术栈：负责评论模块的同事想用 PHP + MySQL 来构建系统，负责用户模块的同事却想用 Golang + PostgreSql 来构建系统。
- 有的模块需要高性能 CPU，有的模块需要大内存，因为不同的模块是耦合在一起的，所以我们的服务器不得不同时具备高性能 CPU，大内存，从而增加了成本。

如何解决此类问题？[康威定律](https://zh.wikipedia.org/wiki/%E5%BA%B7%E5%A8%81%E5%AE%9A%E5%BE%8B)给出了很好的建议：「设计系统的架构受制于产生这些设计的组织的沟通结构」，通俗点说就是：「有什么样的组织架构就会设计出什么样的系统架构」。在本例中，因为不同的团队负责不同的模块，所以很自然的可以通过模块来把系统切分成商城、推荐、评论、用户等几个独立的服务：每个服务有自己独立的版本库和数据库，服务之间通过 RPC 来通信。不同的服务拥有自己的版本库，可以使用适合自己的技术栈和硬件，独立开发独立部署。

一个需要注意的问题是如何确定服务粒度的大小，虽然按照康威定律的描述只要按照组织架构的大小来确定服务的大小即可，但是如何规划一个合理的团队规模呢？实际上并没有一个精确的答案，我们需要按照客观情况来确定一个适合自己的大小适中的服务粒度，过小的粒度会导致服务之间强耦合，过大的粒度则背离了微服务的初衷，Uber 甚至还针对服务粒度大小问题发明了一个[宏服务](https://mp.weixin.qq.com/s/1P_5mMeZQ8YQzybLmjENLg)的概念，有兴趣的读者不妨看看。

![微服务](/img/microservice/02.png)

当我们把单体架构切分成独立的服务之后，原本模块间本地的函数调用变成了服务间远程的 RPC 调用，我们不得不处理服务治理之类的问题，随着微服务数量的增加，问题会变得越来越棘手，好在随着云原生的发展，特别是 [K8S](https://kubernetes.io/) 和 [istio](https://istio.io/) 等技术的成熟，我们的架构可以演化到 [service mesh](https://www.servicemesher.com/) 阶段，通过 sidecar 透明实现服务治理。

![微服务](/img/microservice/03.png)

如果仅仅是把原本模块间本地的函数调用变成了服务间远程的 RPC 调用的话，那么我们的微服务很可能会沦为「[分布式单体](https://skyao.io/talk/202007-microservice-avoiding-distributed-monoliths/)」。问题的症结在于过度使用 RPC，导致服务与服务之间强耦合，解决方法是引入 Event，通过 Event 实现服务与服务的解耦。

看看如何实现下面的业务逻辑：当一个用户注册后，要在商城里给此用户一张优惠券。

- 使用 RPC（强调做什么）：当用户模块创建了一个新用户的时候，通过 RPC 调用商城模块给新此用户一张优惠券，过程中用户模块和商城模块是强耦合的。
- 使用 Event（强调发生了什么）：当用户模块创建了一个新用户的时候，它发出一个 UserCreated 事件，商城模块观察到对应的事件后，给新此用户一张优惠券，过程中用户模块和商城模块是弱耦合的。

实际情况中应该按需求来选择使用 RPC 或则 Event：如果是业务逻辑的实现部分，倾向于选择使用 RPC；如果是业务逻辑完成之后的后续通知部分，强烈建议选择使用 Event。

![微服务](/img/microservice/04.png)

服务部署好了之后，接下来我们还需要考虑如何暴露服务以供前端调用，比如用户浏览某个商品的详情页，内容包括商品数据、以及对应的推荐数据和评论数据，如果直接操作服务的话，那么需要多次查询商品服务、推荐服务、评论服务，并不可取，此时可以加入 [API Gateway](https://microservices.io/patterns/apigateway.html) 充当代理，前端只要请求 API Gateway 一次就可以拿到数据。

![微服务](/img/microservice/05.png)

有了 API Gateway 之后，它可以帮我们完成聚合之类的逻辑。不过有一个问题是前端可能有多种不同的类型，比如 PC 前端，Mobile 前端，它们的业务逻辑不可避免的会有各种各样的差异，如果在 API Gateway 中处理这些差异的话，那么会出现坏味道，为了解决此类问题，我们引入 [BFF](https://microservices.io/patterns/apigateway.html)（Backend For Frontend），每一种前端都有属于自己的 BFF，用来处理专属于自己的业务逻辑，至于 API Gateway，则只处理鉴权，日志等公共业务逻辑。

![微服务](/img/microservice/06.png)

微服务是个极其复杂的概念，本文仅就一些表面问题浅谈一二，其他诸如 [SAGA](https://microservices.io/patterns/data/saga.html) 之类的复杂问题，由于篇幅所限，并未涉猎，大家如果有兴趣的话请自行查阅。

最后把 [Martin Fowler](https://martinfowler.com/) 在 [PoEAA](https://www.martinfowler.com/books/eaa.html) 中提出的[分布式对象第一定律](https://martinfowler.com/bliki/FirstLaw.html)送给大家：不要分布你的对象！套用这个说法的话，不难引申出微服务第一定律：不要使用微服务！虽然话里有一些戏虐的成份，但是它至少告诫我们在面对微服务的时候要怀揣着一颗敬畏的心。
