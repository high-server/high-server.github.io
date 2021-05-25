---
title: "我的应用程序新招了个全能的小秘书"
date: 2021-05-25T15:31:27+08:00
draft: true
#Comment
gitment: true
---


近一年，我的应用程序因为贴近用户需求被推上了热搜 😁 ，不过他也随之变得异常忙碌。来感受下老板和开发童鞋们的吐槽吧~~ 

<!--more-->

“服务器小A，你咋回事儿，这么长时间还不响应，用户是上帝啊！”     
“服务器小B，那么多499，你解释下，这个月奖金还要不要了~”     
………………  

---------

扩容了多次之后，用户投诉是少了不少，可是顺带着也出现了大程序病&#x1F613; 。    
“今年服务成本怎么高了这么多呐？520活动和双11活动的服务资源为啥在不搞活动的时候也在计算成本呐？改改应用程序灵活处理下吧~”    
“下周新版本灰度上线，你们商量下怎么分工，谁负责新版本谁负责旧版本，前提是都得正常服务哈，不能掉链子……”  
“订单模块需要用户模块提供下用户的会员级别，这样才能方便为上帝提供更好的服务呀，可是可是用户模块之前出现过好几次异常，这样可能会拖累订单模块的……”       
………………  

---------

终于，我的应用程序不堪重压，愤起怒吼道&#x1F621; ：“苍天呐，能不能让我专心处理业务逻辑啊，这种‘杂七杂八’的问题又不是我的专业，就不能找专业的人去做吗？”    

有道理啊，给它招个小秘书吧，专门来处理这些”杂七杂八“的事情，说干就干，先出个 JD 吧~

小秘书职责一览：
1. 管理部署&服务发现
2. 负载均衡&自动扩缩容
3. 版本控制&灰度发布&自动部署和回滚
4. 安全验证等

众里寻他千百度，蓦然回首，那人却在灯火阑珊处…………，等等，这不就说的是时代新宠 **Kubernetes** 吗？

## ----------- 我是大大的分界线 ----------   


百科里是这样解释的，**Kubernetes** 简称K8s，是用8代替8个字符“ubernete”而成的缩写。是一个开源的，用于管理云平台中多个主机上的容器化的应用，Kubernetes 的目标是让部署容器化的应用简单并且高效（powerful）,Kubernetes 提供了应用部署，规划，更新，维护的一种机制。     

其中 [容器化](https://mp.weixin.qq.com/s?__biz=Mzg5OTYyMjA1Mg==&mid=2247483728&idx=1&sn=612d8da3e2d2e9ce925b3748dd7a4a86&chksm=c051349bf726bd8d2fe3023ef206d626453312c080a656d2085339a3bc4053addbdc5c21ae83&token=1275028599&lang=zh_CN#rd) 我在之前的一篇文章里具体讲了，这里不做过多阐述，感兴趣可以翻阅。今天主要看看 Kubernetes 这个高级小秘怎么帮助我的应用程序回归业务逻辑自身的？    

为了方便大家理解后面的内容，先来一张K8s官网的架构图,先有个宏观的关键概念及结构的认知    

![arch](/img/k8s-deploy/kubernetes-arch.png)

依据上图，k8s中我们可以根据需求部署不同的资源，Deployment、Service、Headless Service、Load Balancer、Cron Job、DaemonSet、PVC、ServiceAccount、Endpoint、ResourceQuota等(都有相应的controller进行控制，控制器模式)，也可以依据不同的Namespace、Node、Pod、Container进行删选、查看等。这些具体的概念也不是我们今天一起学习的重点，我们就不做过多说明了，后面遇到应用场景真正使用过一次就明白了。     

我们今天主要以一个Deployment为例来看下小秘书是怎么帮助我们的应用程序老大哥解决外围问题的，即我们创建一个nginx的无状态服务deployment的yaml后，k8s是怎么帮助我们达到我们yaml里定义的预期状态的？     
```
$kubectl  apply -f deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2           #运行pod数量
  strategy:
    rollingUpdate:
      maxSurge: 1       #滚动升级时会先启动1个pod
      maxUnavailable: 0 #滚动升级时允许的最大Unavailable的pod个数，不能大于或者等于replicas
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        resources:     #单POD里的CPU、内存资源配置
            limits:
              cpu: "1"
              memory: 1024Mi
            requests:
              cpu: "1"
              memory: 512Mi
          volumeMounts:
            - mountPath: /opt/logs/
              name: logs
          readinessProbe: #重要:kubernetes通过下面接口判断pod启动是否成功，成功了才切流量
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 60
          livenessProbe: #重要:kubernetes通过下面接口定时检测pod是否存活,不存活则会重启
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60
            failureThreshold: 5
            periodSeconds: 60
```

怎么执行呐，我们将其拆解为三大块十小步，建议结合图中的步骤顺序看哈~~     

![flow](/img/k8s-deploy/KubernetesFlow.png)


# 一、Kubectl客户端请求
![flow](/img/k8s-deploy/KubernetesFlow_1.png)

1、**Kubectl** 客户端将yaml资源文件发送给master节点的 **Kube-API-Server** 。Kube-API-Server是提供K8s各类资源对象的增删改查及Watch等HTTP Rest接口，它是集群内各个功能模块之间数据交互和通信的中心枢纽，是整个系统的数据总线和数据中心。  
2、API-Server接收到kubectl客户端的请求后将资源内容存储到 **etcd数据库** 中。etcd数据库是用于持久化存储K8s资源对象的KV数据库。 
# 二、Master节点控制
![flow](/img/k8s-deploy/KubernetesFlow_2.png)

3、**Kube-Controller-manager** 通过list-watch机制监控资源变化并进行响应。k8s中各种资源都是通过控制器模式由相应的controller进行控制的。   
4、**Kube-ReplicaController**(副本控制器，简称RC,高版本已经用ReplicaSet代替，我们这里暂时还用RC)检查数据库变化，创建期望数量的pod实例。     
RC的核心作用就是确保在任何时候集群中某个RC关联的Pod副本数量都保持预设值(上面yaml文件中spec.replicas参数指定的值)。如果发现Pod的副本数量超过预期值，则RC会销毁一些Pod副本；反之，RC会自动创建新的Pod副本，直到符合条件的Pod副本数量达到设定的预期值。同样的，如果某个Pod出现异常它的状态变为不可用时，该Pod会被系统自动回收，管理该Pod的RC将在其他Node上重新创建并运行Pod副本。  
同时，RC 结合 HPA(Horizontal Pod Autoscaler) 可以实现
Pod的自动水平伸缩从而自动完成Pod的扩缩容。流量高峰期资源需求过高时，会自动创建出Pod副本；当资源需求降低时，会自动收缩Pod副本数。    
5、**Kube-ResourceQuotaController** (资源配额控制器)，确保指定的资源对象在任何时候都不会超量占用系统物理资源，避免了由于某些业务进程的设计或实现的缺陷导致整个系统运行紊乱甚至意乱宕机。    
6、**Kube-Scheduler(调度器)** 再次检查数据库变化，监控尚未被分配到具体执行工作节点(Node)的Pod。接下来会分两步选出最优的Node，将pod分配到最优的Node节点上，并更新数据库，记录pod分配情况。第一步根据多种预选策略遍历所有目标Node，筛选出符合要求的候选节点，这个过程中预选策略主要会检测是否有磁盘冲突、Node上的资源是否够用、端口是否占用等。第二部根据在第一步的基础上根据特定的优选策略计算出每个候选节点的积分，积分高者胜出，从而确定最优节点。  
# 三、Node节点控制   
![flow](/img/k8s-deploy/KubernetesFlow_3.png)

7、每个Node上都运行了 **Kubelet**，Kubelet通过list-watch机制监控数据库变化，进而管理Pod后续的生命周期。一旦发现被分配到它所在的节点上运行的那些新Pod，就会在该Node上运行这个新Pod，并对其后续的生命周期负责。   
8、Pod通过 **LivenessProbe** 和 **ReadinessProbe** 两类探针来检查容器的状态。   
LivenessProbe通过在容器内部执行一条命令或者一条http请求等方式来检测容器是否健康并反馈给kubelet。如果LivenessProbe探针检测到容器不健康，则kubelet将删除该容器，并根据容器的重启策略做相应的处理。    ReadinessProbe探针检测容器的就绪状态，即是否启动完成准备接收请求提供服务，这里也就实现了版本平滑升级及回滚等。     
9、k8s集群的每个Node上都会运行 **kuberproxy**，它其实就是一个网络代理，通过iptables或IPVS管理网络通信，如服务发现、负载均衡。例如当有数据发送到主机时，将其路由到正确的Pod或容器。对于从主机上发出的数据，它可以基于请求地址发现远程服务器，并将数据依据特定的负载均衡算法正确路由到合适的服务节点上。  
10、后面还有监控跟踪、日志存储等等。

这么一看，可不吗，相比较于我们裸机部署运营应用程序的时代，相关的自动部署、自动扩缩容、故障转移、负载均衡等等需要解决的问题Kubernetes平台都透明自动的帮我们都解决了呀(参见下表)，大大提升了开发及运维人员工作效率呀，小秘书工作做得棒棒的有木有，太省心了，我的应用程序终于可以聚焦业务本身低头沉浸在自己业务逻辑的搬砖事业中了&#x1F601;……

| |裸机|  K8s|
|:---------------:|:--------------:|:------:|  
|服务上线         |脚本支持&运维辅助      |命令触发平台自动
|测试环境         |人工保障环境一致       |容器镜像保障一致
|版本升级及回滚    |脚本支持&运维辅助      |命令触发平台自动
|热更新           |应用程序代码支持       |平台自动
|灰度测试         |应用程序代码支持       |平台自动
|自动扩缩容       |人工监控&运维辅助      |平台自动
|安全验证         |应用程序代码支持       |平台自动
|流量控制         |代码支持&运维辅助      |平台自动
|负载均衡         |运维辅助             |平台自动
|故障转移         |运维辅助             |平台自动
|日志跟踪及监控    |代码支持&运维辅助      |平台自动

## 总结  
今天我们一起通过一个小故事来理解k8s的使用场景，又结合一个初级的deployment在k8s的部署实现过程来了解了k8s的核心组件的运行原理和机制，希望对你更进一步的了解k8s有所帮助。        
其实k8s还可以结合istio、envoy等服务网格的组件实现流量控制&监控追踪&蓝绿发布等更强大的更能，后面我们再一起探讨~
