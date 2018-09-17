---
author: "trainyao"
date: 2018-09-16 16:08:03
title: kubernetes是怎么工作的以及其架构
linktitle: kubernetes是怎么工作的以及其架构
weight: 10
---

本文简单概括了 kubernetes 里有什么组件，以及它们是如何工作的。

# 一、kubernetes 的组件和架构

一个	kubernetes 集群包含两部分：master 节点和 node 节点。master 节点负责接收管理员对于 kubernetes 资源对象的修改，并根据这些修改，对 node 节点进行配置和调度。node 节点是 kubernetes 集群内的计算节点，负责给部署在上面的业务应用提供计算以及储存能力。

kubernetes 里有以下组件：Kubernetes API Server、Controll Manager、Scheduler、kubelet 和 kube-proxy。

其中，Kubernetes API Server、Controll Manager、Scheduler 运行在 master节点上。而 kubelet 和 kube-proxy 运行在 node 节点上。

这些组件分别负责 kubernetes 运行的不同职责：

- Kubernetes API Server：kubernetes 集群的管理入口，它提供 HTTP REST 接口给 kubernetes 用户进行集群的配置，这些配置将会保存在 master 节点运行的 etcd 里面，供其他组件查阅。

- Controll Manager：负责监听 etcd 变化并作出对应的动作。Controll Manager 是一系列 manager 的统称，其中有以下这些manager：
	- Replication Manager：负责 node 节点的 pod 管理。
	- Node Controller：负责 node 节点管理。
	- Namespace Controller：负责管理集群内的命名空间。
	- ResourceQuota Controller：负责针对 namespace、pod、容器这三个维度的资源限制管理。
	- ServiceAccount Controller：负责管理集群内访问的账号。
	- Token Controller：负责允许访问 kubernetes 集群的 token。
	- Service Controller：负责管理集群内的 Service。
	- Endpoint Controller：负责管理集群内的 Endpoint。

- Scheduler：负责接收 Replication Controller 的指挥，对要部署在 node 上的 pod 进行调度。Scheduler 会根据调度算法以及调度策略，调度 pod 在哪个 node 上部署。

- kubelet：node 节点上运行与 master 节点进行交互的进程。负责监控集群 etcd 内配置的变动，对本 node 上的 pod 进行管理。
- kube-proxy：node 上的网络 sidecar，负责 node 上的容器的网络请求以及外部网络请求的代理。	kubernetes 的 service 概念以及负责均衡就是依靠 kube-proxy 实现的。

# 二、场景分析

根据一些 kubernetes 的使用场景，来说明各个组件的协作过程。

- 创建 namespace
- 创建 deploymentA
- 创建 serviceA
- 将 创建 serviceA 的 nodeport 配置，并从外部访问service
- 创建 serviceB 以及 deploymentB
- deploymentA 通过 service 名访问 deploymentB
(未完待续)


