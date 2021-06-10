---
title: "从iptables谈ServiceMesh流量拦截"
date: 2021-06-10T14:38:48+08:00
draft: false
categories:
  - "架构技术"

#Comment
gitment: true

---

最近研究学习 Kubernetes 和 ServiceMesh 过程中都看到了 **iptables** 发挥重要作用独挡一面的场景。     
Kubernetes 中 iptables 作为 kube-proxy 里控制流量转发的核心模式，通过在目标 node 的 iptables 中增加一些自定义链对流经到该 node 的数据包做DNAT和SNAT操作以实现路由、负载均衡和地址转换。     
ServiceMesh 服务网格中 Istio 通过 init 容器（启动命令为istio-iptables）给 Sidecar 容器即 Envoy 代理做初始化，设置 iptables 端口转发，从而实现流量劫持&转发控制等服务治理相关。

<!--more-->


But，仔细搜索大脑海马体，发现对 iptables 的认知并不是那么清晰，所以一起重新梳理一遍吧~~

## 一、iptables 基础

   提起 iptables 相信大家的第一反应是--防火墙，不知道你们是不是，反正我的第一反应是这样的，偷笑……     
   但其实 Linux 中包过滤防火墙全称为 Netfilter/Iptables。位于内核空间的 Netfilter 才是防火墙真正的安全框架，它可以根据规则实现数据包的过滤、网络地址转换、内容修改等。Iptables 是位于用户空间的一个命令行工具，用来操作 Netfilter，对 NetFilter 中的规则进行增删改查，从而将用户的安全设定同步进内核空间的 Netfilter。

   后面用 iptables 代替 Netfilter/Iptables。
   我们先用一个思维导图来看下 iptables 的核心概念（按优先级画的表，实际上使用频率及场景的话 filter 表和 nat 表使用的更多些）
   ![iptables](/img/iptables/iptables.png)
   
   根据思维导图我们可以看到 iptables 中有 **五条链** -- PREROUTING路由前/INPUT流入/FORWARD转发/OUTPUT流出/POSTROUTING 路由后 和 **四张表** -- Raw/Mangle/Nat/Filter，各个表和链中又可以设定多条规则，从而对数据包进行控制。

   里面有几个名词需要解释下：   
   **链(Chain)**：之所以先解释链是因为内核源码里最直接的 iptables 线索是通过链来体现的（思维导图里为了按功能区分先画的表）。链其实是按数据包流向划分的一些钩子函数 HOOK，内核源码里搜索 NF_HOOK 可以看到相关内容，根据数据包所处的不同网络协议层，都有不同的链 HOOK，如 ARP 层、Bridge 层、IP 层、Application 层，具体各层的 HOOK 细节可以参考[NetFilter hooks](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)
   ![kernel_nf_hooks](/img/iptables/kernel_nf_hook.png)
   **表(Table)**：表是按功能化分的规则集合，如nat表主要负责网络地址转换，filter表主要负责过滤，mangle表主要负责修改数据包标记等，就如同一个餐厅里的服务员、大厨、配菜员、洗消工人、保安等分工不同。每条链中根据功能不同可以使用特定的几个或全部表的规则。     
   **规则(Policy)**：按指定的条件来匹配数据包，匹配成功后执行相应的通过/丢弃等动作处理。 

   打个比方，有一家高档的西餐厅：   
   **进门时**，服务员会先审视一遍顾客，因为餐厅规定需着正装进入所以光膀子大裤衩人字拖的就被拒之门外了，此时顾客还不服气大吵大闹的，这时候保安只能按违反公共环境保持安静的要求把他们带走了。另外一位顾客穿着得体但是带了宠物，服务员指着旁边告示牌上写的告示礼貌的提醒顾客将宠物暂存在了前台准备的宠物笼中。   
   **第二步开始点餐**，顾客点了一份七分熟的牛排和一份鱼香肉丝，等会儿，这是西餐厅不卖鱼香肉丝啊，西餐厅出现鱼香肉丝会影响餐厅形象的，服务员非常有礼貌的拒绝了顾客鱼香肉丝的需求并告知顾客鱼香肉丝可以去旁边的川菜馆享用，同时正常下单了牛排的需求。     
   **第三步制作阶段**，配菜员配菜，需按餐厅宣传的牛排肉质的要求及分量进行配菜，然后交给大厨进行烹饪，这里注意牛排要求七分熟，所以大厨特别认真的盯紧火候生怕过大或者过小，毕竟火候过了或者欠了都不满足顾客需求，都会被顾客投诉。     
   **第四步买单阶段**，顾客享用完美味的餐食后，离开餐桌准备买单，这时服务员会负责引导顾客到达收银处进行买单，同时洗消工人进行餐桌餐具及厨余垃圾的洗消工作。 

   结合这个场景屡一下链、表和规则的关系：   
   上客阶段（**链1**）：服务员（**表1**，保障餐厅形象）-- **规则1**要求穿着得体、规则2要求禁止携带宠物；保安（**表5**，维持餐厅秩序） -- 规则1禁止大吵大闹  
   点餐阶段（**链2**）：服务员（**表1**，保障餐厅形象）-- 拒绝提供非西餐食品    
   制作阶段（**链3**）：配菜员（**表2**，保障餐厅食材品质）-- 严格筛选高品质肉类及蔬菜；大厨（**表3**，保障餐厅食物口感）-- 严格控制火候、调味剂分量及每个步骤时长等    
   买单阶段（**链4**）：服务员（**表1**，保障餐厅形象）、洗消工人（**表4**，保障餐厅卫生环境）-- 都有具体的卫生要求标准       

   现在是不是清楚些了，iptables中链是主线条，然后每条链按需要执行特定功能表中的一条或多条规则，如上客阶段需要服务员表中的规则和保安表中的规则。

   iptables中的四张表按优先级raw –> mangle –> nat –> filter执行。然后具体的规则按照从前往后的优先级执行，要求越严格的规则应该放在越靠前面。     
   思维导图中对四张表分别做了简要的功能概述，接下来我们看下五条链以及他们根据功能需要用到哪些表的规则。     
   ![iptables_chain](/img/iptables/iptables_chain.png)

   现在我们对数据包的流向及相应的表和链的控制有了个大概的了解，如果还需要更详细的逐步的查看请参考[iptables四个表五条链](https://blog.csdn.net/longbei9029/article/details/53056744)。

## 二、iptables 命令使用 

   官方定义的命令格式如下：
   ```
   iptables -t 表名 <-A/I/D/R/L/F> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP> --sport 源端口 <-d 目标IP> --dport 目标端口 -j 动作  

   -t 指定表名，未指定表名时默认为 Filter 表
   -A和-I都是添加规则，-A增加的规则放在现有规则的最后，-I添加的规则放在规则号指定的位置，该位置原先的规则往后顺位。
   -D 删除规则号指定的规则
   -R 替换规则号指定的规则
   -L 查看相应的规则
   -F 清楚某条链或者表的规则
   -i/o 指定输入和输出的网卡
   -p 指定数据包协议，如 tcp、udp、icmp 等，这里支持简单的表达式，如 -p !tcp 去除 tcp 外的所有协议
   -s和-sport分别指定数据包源 IP 地址及端口
   -d和-dport分别指定数据包目标 IP 地址及端口
   -j 指定前述的参数匹配上数据包以后执行的动作。常用的处理动作包括 ACCEPT 放行、REJECT 拒绝、DROP 丢弃、REDIRECT 重定向、DNAT 修改目的 IP 及端口、SNAT 修改源 IP 及端口等等
  
```
   另外还有一些高级的参数，如参数 tcp-flags (只过滤 TCP 中的一些包，比如 SYN 包，ACK 包，FIN 包，RST 包等等)、参数 limit 限制数据包的平均流量、参数 state 过滤特定状态(如Established、Invalid 等)的数据包，类似的参数还有不一一列举了。     


   下面结合一些实际应用场景看看具体怎么使用吧~~

   1.查看服务器上 nat 表的所有规则
   ```
[@hbhly_65_203 ~]# iptables -L -n -v -t nat
Chain PREROUTING (policy ACCEPT 11 packets, 2250 bytes)
 pkts bytes target     prot opt in     out     source       destination

Chain INPUT (policy ACCEPT 11 packets, 2250 bytes)
 pkts bytes target     prot opt in     out     source       destination

Chain OUTPUT (policy ACCEPT 35 packets, 2838 bytes)
 pkts bytes target     prot opt in     out     source       destination

Chain POSTROUTING (policy ACCEPT 35 packets, 2838 bytes)
 pkts bytes target     prot opt in     out     source       destination
```
 这里再次看到了 nat 表只包含了 PREROUTING/INPUT/OUTPUT/POSTROUTING 四条链的规则，不包含 FORWARD 链的规则。

 2.禁用SSHD默认的22端口
 ```
 iptables -t filter -A INPUT -p tcp --dport 22 -j DROP
```
 3.只允许特定网段10.160.0.0/16访问本机的10.160.100.1的SSHD(22端口)服务
```
 #设置默认的drop，再允许特定的网段进入和出去
 iptables -P INPUT DROP
 iptables -P OUTPUT DROP
 iptables -P FORWARD DROP

 iptables -t filter -A INPUT -s 10.160.0.0/16 -d 10.160.100.1 -p tcp --dport 22 -j ACCEPT
 iptables -t filter -A OUTPUT -s 10.160.100.1 -d 10.160.0.0/16 -p tcp --dport 22 -j ACCEPT
```
 4.过滤掉状态有问题的http包。只允许http80端口且限定连接状态为Established和Related的数据包
 ```
 iptables -A INPUT -p tcp  --sport 80 -m state --state ESTABLISHED,RELATED -j ACCEPT
```
 5.开启儿童上网模式，星期一到星期五的8:00-21:00禁止游戏相关网页”game“
```
iptables -I FORWARD -s 192.168.0.0/24 -m string --string "game" -m time --timestart 8:00 --timestop 21:00 --days Mon,Tue,Wed,Thu,Fri -j DROP
```
 6.生产环境mysql数据库仅允许内网特定ip访问
 ```
 iptables –A INPUT –s 10.160.41.1 –p tcp –dport 3306 –j ACCEPT
```
 7.将目的IP为10.160.132.55且目的端口为9090的我们做DNAT修改目标地址处理，重定向到10.162.37.1:8080 
 ```
 iptables  -A INPUT -d 10.160.132.55 -p tcp --dport 9090 -j DNAT --to 10.162.37.1:8080
```
 8.拦截所有入站tcp80端口和8080端口数据包重定向到某个代理服务的15001端口进行统一处理
 ```
 iptables -A INPUT -p tcp --dport 80,8080 -j REDIRECT --to-ports 15001
 ```
 ps:这里已经看到 ServiceMesh 服务网格中 sidecar 模式的 Envoy 代理里实现流量拦截从而进行统一的服务治理的一点身影了~~~


## 三、ServiceMesh 中 iptables 的发挥

   ServiceMesh 中 iptables 怎么发挥作用的呐？
   Istio 通过为 Pod 注入 Init 容器来为 Pod 设置 iptables 规则进行端口转发。
Init 容器是一种专用容器，它在应用程序容器启动之前运行，用来包含一些应用镜像中不存在的实用工具或安装脚本。
它的启动命令是 
```
istio-iptables -p 15001 -z 15006 -u 1337 -m REDIRECT -i '*' -x "" -b '*'
```
感兴趣可以具体查看istio中源码https://github.com/istio/istio/blob/master/tools/istio-iptables，这里不再单独拿出来说了。

istio中iptables命令格式如下：
```
istio-iptables -p PORT -u UID -g GID [-m mode] [-b ports] [-d ports] [-i CIDR] [-x CIDR] [-h]

  -p: 指定重定向所有 TCP 流量的 Envoy 端口（默认为 $ENVOY_PORT = 15001）
  -u: 指定未应用重定向的用户的 UID。通常，这是代理容器的 UID（默认为 $ENVOY_USER 的 uid，istio_proxy 的 uid 或 1337）
  -g: 指定未应用重定向的用户的 GID。（与 -u param 相同的默认值）
  -m: 指定入站连接重定向到 Envoy 的模式，“REDIRECT” 或 “TPROXY”（默认为 $ISTIO_INBOUND_INTERCEPTION_MODE)
  -b: 逗号分隔的入站端口列表，其流量将重定向到 Envoy（可选）。使用通配符 “*” 表示重定向所有端口。为空时表示禁用所有入站重定向（默认为 $ISTIO_INBOUND_PORTS）
  -d: 指定要从重定向到 Envoy 中排除（可选）的入站端口列表，以逗号格式分隔。使用通配符“*” 表示重定向所有入站流量（默认为 $ISTIO_LOCAL_EXCLUDE_PORTS）
  -i: 指定重定向到 Envoy（可选）的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量。空列表将禁用所有出站重定向（默认为 $ISTIO_SERVICE_CIDR）
  -x: 指定将从重定向中排除的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量（默认为 $ISTIO_SERVICE_EXCLUDE_CIDR）。
```
启动之后netstat查看就发现相关端口了     
![netstat](/img/iptables/netstat.png)

咱们上面用到的那行启动命令，直白的讲就是**让 Envoy 代理可以拦截所有的进出 pod 的流量，所有入站（inbound）流量重定向到 15006 端口（sidecar），再拦截应用容器的出站（outbound）流量经过 sidecar 处理（通过 15001 端口监听）后再出站**。厉害了，大有一种不管什么牛鬼蛇神都要来我这里报个到，我来决定你们下一步的走向的架势，就像windows的电脑管家拿到了绝对控制权还不是想弹个啥窗就弹个啥，所有流量都拦截了再进行流量控制、熔断限流、负载均衡等服务治理就大有可为了。  
这个命令其实是 iptables 命令的变形，但是换汤不换药，根本原理是一样的，最终也是生成了一系列的 iptables 规则，可以服务器上自己验证下，为了方便标注每一步的过程我按代码格式贴出来了~

```
$ iptables -t nat -L 
# PREROUTING 链：将所有入站 TCP 流量跳转到 ISTIO_INBOUND 链上，这里不要好奇怎么又多出来了个上面没讲的链，iptables是可以自定义链的哈，k8s和istio中都有大量的自定义链
Chain PREROUTING (policy ACCEPT)
target         prot opt     source               destination
ISTIO_INBOUND  tcp  --      anywhere             anywhere

# INPUT 链：没有规则，正常跳转到OUTPUT链
Chain INPUT (policy ACCEPT)
 target     prot opt   source               destination

# OUTPUT 链：将所有出站数据包跳转到 ISTIO_OUTPUT 链上
Chain OUTPUT (policy ACCEPT)
target        prot opt    source               destination
ISTIO_OUTPUT  tcp  --     anywhere             anywhere

# POSTROUTING 链：没有规则
Chain POSTROUTING (policy ACCEPT)
target        prot opt    source               destination

# ISTIO_INBOUND 链：将所有目的地为 9080 端口的入站流量重定向到 ISTIO_IN_REDIRECT 链上
Chain ISTIO_INBOUND (1 references)
target             prot opt   source               destination
ISTIO_IN_REDIRECT  tcp  --    anywhere             anywhere      tcp dpt:9080

# ISTIO_IN_REDIRECT 链：将所有的入站流量跳转到本地的 15001 端口，至此成功将流量拦截到了Envoy代理
Chain ISTIO_IN_REDIRECT (1 references)
target     prot opt    source               destination
REDIRECT   tcp  --     anywhere             anywhere             redir ports 15001

# ISTIO_OUTPUT 链：这里需要注意区分两种情况，所有非 localhost 的流量全部转发到 ISTIO_REDIRECT。为了避免流量在该 Pod 中无限循环，所有到 istio-proxy 用户空间的流量(启动命令行中通过参数有进行过滤)都返回到它的调用点中的下一条规则即 OUTPUT 链，因为跳出 ISTIO_OUTPUT 规则之后就进入下一条链 POSTROUTING。如果目的地非 localhost 就跳转到 ISTIO_REDIRECT；如果流量是来自 istio-proxy 用户空间的，那么就跳出该链，返回它的调用链继续执行下一条规则（OUPT 的下一条规则，无需对流量进行处理）；所有的非 istio-proxy 用户空间的目的地是 localhost 的流量就跳转到 ISTIO_REDIRECT
Chain ISTIO_OUTPUT (1 references)
target          prot opt     source               destination
ISTIO_REDIRECT  all  --      anywhere            !localhost
RETURN          all  --      anywhere             anywhere     owner UID match istio-proxy
RETURN          all  --      anywhere             anywhere     owner GID match istio-proxy	
RETURN          all  --      anywhere             localhost
ISTIO_REDIRECT  all  --      anywhere             anywhere

# ISTIO_REDIRECT 链：将所有流量重定向到 Envoy的 15001 端口
Chain ISTIO_REDIRECT (2 references)
target     prot opt    source               destination
REDIRECT   tcp  --     anywhere             anywhere      redirect ports 15001
```
结合官网给的过程图更容易理解些，特别详细了，顺着不同颜色的箭头按数字顺序看就OK，我就不再献丑了~~

![istio-iptables](/img/iptables/istio-iptables.png)
怎么样，看似高深莫测的 ServiceMesh 服务网格经由这么一看是不是感觉没那么神秘了呐，当然也不是说这就是它的全部了，iptables 只是其流量拦截的基本技术面，其他内容后面有空可以再一起深挖。   
与此同时，不知道你有没有发现问题，与传统服务模式相比，这种所有数据包要到达目的地每一步都要经过一系列的拦截代理转发，势必会对服务的性能及负载等产生或多或少的影响，这也是网格技术如此火热却长期有价无市没有普及应用的一个重要原因，当然还有其他的一些原因，同时社区也在竭尽全力的去优化这部分，相信未来可期。    

## 四、总结
本文中我们一起梳理了 iptables 的基础概念，如各种规则、四个表、五条链等，又一起根据实际场景探讨了 iptables 命令的使用，而后进一步将其结合时下火热的 ServiceMesh 技术中 Envoy 代理进行流量拦截的原理对 sidecar 模式进行了理解，Kubernetes 中也有 iptables 的深入应用，后面我们找时间再具体谈论这块儿，敬请期待~~
