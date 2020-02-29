---
author: "trainyao"
date: 2018-09-18
title: Kubernetes 使用 IPVS/LVS 作为网络服务代理
linktitle: Kubernetes 使用 IPVS/LVS 作为网络服务代理
categories: ["翻译"]
weight: 10
---

**本文翻译自：[https://cloudnativelabs.github.io/post/2017-05-10-kube-network-service-proxy/](https://cloudnativelabs.github.io/post/2017-05-10-kube-network-service-proxy/)**

服务是 Kubernetes 的一个抽象概念。服务将一组逻辑上的、提供相同功能的 pod 组合在一起。一个在 Kubernetes 内的服务可以是多种类型的，比如 ‘ClusterIP’ 和 ‘NodePort’ 类型实现服务发现和负载均衡。这些服务的类型的实现都需要一个运行在各个 Kubernetes 集群节点的服务代理。Kubernetes 有一种名为 ‘kube-proxy’ 的代理，是基于 iptable 实现的。虽然，kube-proxy 提供了一种开箱即用的代理解决方案，它并不是一种最优的方案。在这篇文章里，我们会深入研究另一种 Kubernetes 里网络服务代理的实现：kube-router，它是基于 Liunx IPVS 的。我们还会讨论基于 iptables 的 kube-proxy 的好处与不足，以及将它与基于 IPVS/LVS 实现的 Kubernetes 网络服务代理进行对比。

请看下面的 demo，感受 IPVS 实现的 kube-router 作为 Kubernetes 服务代理的效果。

（此处查看[原文](https://cloudnativelabs.github.io/post/2017-05-10-kube-network-service-proxy/)可以看到 asciinema demo，如果不能访问，asciinema 地址在[这里](https://asciinema.org/a/120312)）

下面我们来深入研究一下细节。

### ClusterIP 和 NodeProt 类型的服务

在服务的生命周期里，每个 ‘ClusterIP’ 和 ‘NodePort’ 服务会被分配一个唯一的 IP 地址（称为 ClusterIP）。Kubernetes 会配置 Pods 通过 ClusterIP 与服务进行交互，而且 pods 发送至 service 的请求会被自动地负载均衡到 service 的成员 pods 上。另外，Kubernetes master 节点会为 ‘NodePort’ 类 service 分配一个标志配置范围（默认是 30000 至 23767）的端口，然后每个 node 节点会将该 service 的请求代理到这个端口上。

### IPVS

IPVS (IP Virtual Server) 在 Linux 内核实现了传输层（L4）负载均衡。运行在主机上的 IPVS 表现为一个负载均衡，为集群内真实的服务器提供负载均衡功能。它可以将 TCP/UDP 请求指向到真正的服务器，并使这些服务器表现的像是一个拥有单独 IP 地址的虚拟服务。IPVS 在 Linux 内核层面支持丰富的连接调度算法（轮询、带权重的轮询、最小连接数等等）。由于是内核实现，IPVS 速度很快，并且提供了不同的 IP 负载均衡技术（NAT、TUN、LVS/DR）。

### 基于 IPVS 实现的 Kubernetes 服务代理

我们刚刚已经了解到，每个在 node 节点上的 ‘ClusterIP’ 或 ‘NodePort’ 类 Kubernetes service 都需要一个负载均衡器和服务代理。Kubernetes 的 service 和 endpoint 概念，实际上可以对应到 IPVS 的虚拟服务和服务器概念。这部分工作交给 IPVS 后，Kubernetes 就很容易为 ‘ClusterIP’ 或 ‘NodePort’ 类 service 提供服务代理，只需一个运行在每个集群上的 node 节点的代理，由代理监控 Kubernetes API server 的 Service 和 Endpoint 的变动，并且映射到对应的 IPVS 配置和状态上。

### 基于 IPVS 实现的 Kube-router 服务代理实现

Kube-router 基于 IPVS，为 ‘ClusterIP’ 和 ‘NodePort’ 类 service 实现了服务代理。Kube-router 监听 Kubernetes API server 的 service 和 endpoint 变化（新增/删除/修改），然后配置 IPVS 的虚拟服务和服务器。虽然映射 service 和 endpoint 到 IPVS 的虚拟服务和服务器是很直接的，这仍然有一些问题需要解决：

#### cluster ip

通常 Cluster IP 是不能路由的（不是 Kubernetes 的硬性约束，使用者可以绑定这些 IP 到非集群的路由环境）。一个 Cluster IP 会被 Kubernetes 集群的所有 node 节点共享。至少 node 节点上的 pod 需要能访问这些 ClusterIP。Kube-router 在 node 上提供专门用作分配 cluster IP 的假接口（dummy interfate），解决了这个问题。一旦 IP 被分配到 node 上，node 上的 pod 就能够访问到这个 cluster IP。因此从 node 外部访问 cluster IP 的流量就不会被默认的丢弃掉。

#### 反向流量

Kube-router 使用 IPVS NAT 模式作为负载均衡技术。就像基于 DNAT 的负载均衡机制一样，反向路径流量需要通过负载均衡器进行端到端功能。有两种来源的数据路径需要满足这种需求。对于来自 pod 和 node 的流量，只有目标 NAT（destination NAT）会生效。由于 kube-router 使用基于主机的路由实现 pod 对 pod 的网络连接，反向路径流量会流经本身产生流量的那个 node。另外一种来源，来自集群外部的流量，它们访问 node 上的端口，由于需要流量伪装，SNAT会生效。在反向路径上，流量通过与 SNAT 相同的 node。

#### ipvs conntrack
为了保证性能，IPVS 使用本身的简单而快速的连接跟踪功能，而不是 netfilter 连接跟踪。Kube-router 也启用了 IPVS 的连接跟踪功能。

### Kube-proxy
Kube-proxy 是一个运行在每个 Kubernetes 集群节点的为了代理服务，它用轮询的策略，代理传输层回话到 service 的 VIP（service 的 cluster IP）或者 node port 上，以请求 service 后面的 pod。Kube-proxy 是基于 Linux 的 iptables 功能实现的。

#### 优点：

- 利用 iptable，SNAT、DNAT、端口映射功能都可以实现。可以在数据包传输的不同阶段对数据包进行处理（路由前、路由后、输入、传输中、输出等等）。
- 支持端口范围

#### 不足：

- 负载均衡实现比较晦涩，如果不知道 kube-proxy 是如何使用 iptables 作为负载均衡的话，很难进行故障排除。
- 不是一个真正的负载均衡器，只是一个具有轮询机制的数据包转发者
- 没有负载均衡机制
- 随着 service 数量上升，iptables 的性能会下降。对于数据处理部分，service 的数量越多，意味着单个数据包需要匹配一长串 iptable 规则。对于控制部分，插入和删除这些规则的时候性能也很差。

### 基于 IPVS/LVS 的服务代理

#### 优点：
- 对比于 iptable 的规则列表匹配，IPVS 基于哈希匹配。Service 和 endpoint 数目上升并不会影响 IPVS 的匹配速度。
- 最大的好处是通过 ipvsadm 工具，非常容易校验配置。
- 在 Linux 内核 里存在已久，并且被广泛使用。

#### 不足：
- 仍然需要 iptable tweeks 来实现伪装
- 直接路由不能处理端口重映射，所以速度最快的直接路由负载均衡算法不能使用。

### 结论

一个基于 IPVS 的服务代理对于 Kubernetes 部署是非常使用的。它很快，很多人使用，并且根据实际测试很容易进行校验和故障排除。除非有某些特殊要求（比如支持端口范围）只能通过基于 iptables 的 kube-proxy 才能满足，基于 IPVS 的服务代理是一个更好的选择。由于我们建立在坚实的基础之上，为我们解决了重要的问题，我们的解决方案可以少些几百行代码，而且还更不容易出错。