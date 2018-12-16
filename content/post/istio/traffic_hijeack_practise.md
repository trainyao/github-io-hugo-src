---
author: "trainyao"
date: 2018-12-08 18:25:54
title: istio 流量劫持实验
linktitle: istio 流量劫持实验
weight: 10
---

istio 是通过给每个生产 pod 注入一个 envoy sidecar 进行流量劫持的。注入后，流入 pod 的流量和 pod 请求的流量都会经过 sidecar 进行路由。

这样的流量劫持是通过 sidecar pod 和生产 pod 共享一个网络命名空间，然后设置 iptable 规则实现的。

PS1：下面的步骤前提是在一个 k8s 集群内操作，并且已经安装了 istio）
PS2：实验步骤是基于宋大的博文上实验的，可以先看看宋大的两篇博文：[用 vagrant 和 virtualbox 搭建一个 k8s 集群](https://github.com/rootsongjc/kubernetes-vagrant-centos-cluster) [理解 Istio Service Mesh 中 Envoy 代理 Sidecar 注入及流量劫持](https://jimmysong.io/posts/envoy-sidecar-injection-in-istio-service-mesh-deep-dive/)

# 1.sidecar 是如何注入的呢？

比如我们要部署一个服务，首先要编写一个关于 service 和 deployment 的 yaml 文件（拿 istio 官网的 bookinfo 为例子）：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: productpage
  labels:
    app: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: productpage-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: productpage
        version: v1
    spec:
      containers:
      - name: productpage
        image: istio/examples-bookinfo-productpage-v1:1.5.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
```

这个是 productpage 服务的 yaml 的内容。那么调用 `istioctl kube-inject -f productpage.yaml` 后会发生什么呢？

```shell
$ istioctl kube-inject -f productpage.yaml
---
##################################################################################################
# Productpage services
##################################################################################################
apiVersion: v1
kind: Service
metadata:
  name: productpage
  labels:
    app: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  name: productpage-v1
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      annotations:
        sidecar.istio.io/status: '{"version":"55c9e544b52e1d4e45d18a58d0b34ba4b72531e45fb6d1572c77191422556ffc","initContainers":["istio-init"],"containers":["istio-proxy"],"volumes":["istio-envoy","istio-certs"],"imagePullSecrets":null}'
      creationTimestamp: null
      labels:
        app: productpage
        version: v1
    spec:
      containers:
      - image: istio/examples-bookinfo-productpage-v1:1.5.0
        imagePullPolicy: IfNotPresent
        name: productpage
        ports:
        - containerPort: 9080
        resources: {}
      - args:
        - proxy
        - sidecar
        - --configPath
        - /etc/istio/proxy
        - --binaryPath
        - /usr/local/bin/envoy
        - --serviceCluster
        - productpage
        - --drainDuration
        - 45s
        - --parentShutdownDuration
        - 1m0s
        - --discoveryAddress
        - istio-pilot.istio-system:15007
        - --discoveryRefreshDelay
        - 10s
        - --zipkinAddress
        - zipkin.istio-system:9411
        - --connectTimeout
        - 10s
        - --statsdUdpAddress
        - istio-statsd-prom-bridge.istio-system:9125
        - --proxyAdminPort
        - "15000"
        - --controlPlaneAuthPolicy
        - NONE
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: INSTANCE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: ISTIO_META_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ISTIO_META_INTERCEPTION_MODE
          value: REDIRECT
        image: docker.io/istio/proxyv2:0.8.0
        imagePullPolicy: IfNotPresent
        name: istio-proxy
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
        securityContext:
          privileged: false
          readOnlyRootFilesystem: true
          runAsUser: 1337
        volumeMounts:
        - mountPath: /etc/istio/proxy
          name: istio-envoy
        - mountPath: /etc/certs/
          name: istio-certs
          readOnly: true
      initContainers:
      - args:
        - -p
        - "15001"
        - -u
        - "1337"
        - -m
        - REDIRECT
        - -i
        - '*'
        - -x
        - ""
        - -b
        - 9080,
        - -d
        - ""
        image: docker.io/istio/proxy_init:0.8.0
        imagePullPolicy: IfNotPresent
        name: istio-init
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
          privileged: true
      volumes:
      - emptyDir:
          medium: Memory
        name: istio-envoy
      - name: istio-certs
        secret:
          optional: true
          secretName: istio.default
status: {}
---
```

可以看到，除了我们定义的 productpage 容器外，istioctl 命令帮我们在 deployment 的配置里面加上了一个 pod 的 initContainer，并且加上了一个叫做 istio-proxy 的容器。

从这个 initContainer 和 istio-proxy 的 dockerfile 和启动参数看出，initContainer 是负责设置 iptable 规则的，而 istio-proxy 则是负责流量路由的容器。

至此，sidecar 注入完成。

# 2. iptable 规则和 istio-proxy 是如何劫持流量的？

先进入 istio-proxy 容器，查看一下 iptable 规则。
```shell
$ kubectl exec -it productpage-v1-6cbd9dff5d-vv8dk bash -c istio-proxy
istio-proxy@productpage-v1-6cbd9dff5d-vv8dk:/$
istio-proxy@productpage-v1-6cbd9dff5d-vv8dk:/$ iptables -nL -t nat -v
iptables v1.6.0: can't initialize iptables table `nat': Permission denied (you must be root)
Perhaps iptables or your kernel needs to be upgraded.
istio-proxy@productpage-v1-6cbd9dff5d-vv8dk:/$ sudo -i
mesg: ttyname failed: Success
root@productpage-v1-6cbd9dff5d-vv8dk:~# iptables -nL -t nat -v
iptables v1.6.0: can't initialize iptables table `nat': Permission denied (you must be root)
Perhaps iptables or your kernel needs to be upgraded.

```

看不到啊，网上搜索了后发现是 istio-proxy 权限不足。回到上文 `istioctl kube-inject` 生成的 yaml 文件看到，"privileged=false" 限制了 istio-proxy 容器的权限。可以用 `kubectl edit` 命令编辑为 true，然后保存重新部署 productpage 服务。

再次执行:
```shell
trainyaodeMacBook-Pro:kubernetes trainyao$ kubectl exec -it productpage-v1-5ddc866b9-6567x bash -c istio-proxy
istio-proxy@productpage-v1-5ddc866b9-6567x:/$
istio-proxy@productpage-v1-5ddc866b9-6567x:/$ sudo -i
mesg: ttyname failed: Success
root@productpage-v1-5ddc866b9-6567x:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 22 packets, 1987 bytes)
 pkts bytes target     prot opt in     out     source               destination
    3   180 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 22 packets, 1987 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
    3   180 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
    0     0 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001
root@productpage-v1-5ddc866b9-6567x:~#
```

看到了。

initContainer 设置了 PREROUTING 和 OUTPUT 规则（在包开始进入用户进程和开始离开用户进程时的 iptable 规则），并且新建了与 istio 相关的规则链 ISTIO_INBOUND、ISTIO_OUTBOUND 和 ISTIO_REDIRECT。ISTIO_REDIRECT 链主要是将流量重定向到 15001 端口上，这个是 envoy 代理监听的端口。而ISTIO_INBOUND、ISTIO_OUTBOUND 像是入口函数一样，与进入 istio、离开 istio 相关的逻辑都在这个规则链里，在必要的时候将流量重定向到 ISTIO_REDIRECT 链。那么在那里调用他们呢？就要看 PREROUTING、和 OUTPUT 规则链了。

在 PREROUTING 链和 OUTPUT 链的规则可以看到，任意的接口，任意的来源和目标的 tcp 包都会被重定向到 ISTIO_INBOUND 和 ISTIO_OUTBOUND 链，而 PREROUTING 链和 OUTBOUND 链刚好是进入和离开用户进程的最开始（可参考 iptables 的），由此 istio 实现了流量劫持。

那么 ISTIO_INBOUND 和 ISTIO_OUTBOUND 链做了什么逻辑呢？

ISTIO_INBOUND 链将任何目标端口为 9080 的 tcp 包都重定向到 envoy；ISTIO_OUTBOUND 链则将`来自 1337 uid/gid 的包和目标为 127.0.0.1的包`直接返回，不进入 envoy 外，其余的包都路由到 envoy，让 envoy 控制包的转发。1137 uid/gid 即 istio-proxy 用户和其所在的组，该用户正是启动 envoy 进程的用户。这两个配置是防止从 envoy 进程出来的包，继续被重定向进入 envoy 进程，从而使流量无限循环。目标为 127.0.0.1 的包比较好理，这种目标地址的包不需要处理。

# 3. 被劫持的流量都是怎么走的？
分析完了 istio 通过 iptable 的设置来劫持流量，可以分析被劫持的流量都是怎么经过这些规则的。

首先，流量都有可能从哪里进入呢？
- 从 istio 定义的 IngressGateway 进入（例子里表现为 k8s 里的 nodeport）
- 从 其他 pod 容器 curl service 的名字的流量
- 从 其他 pod 容器 curl service 的 IP 进入的流量
- 从 其他 pod 容器的 sidecar 容器 curl service 的名字的流量
- 从 其他 pod 容器的 sidecar 容器 curl service 的 IP 进入的流量
- 从 productpage pod curl localhost 进入的流量
- 从 productpage istio-proxy sidecar 容器 curl localhost 进入的流量

下面集合 tcpdump 工具，跟踪一下不同的流量都是怎么被路由的。

实验里面用了 istio 的官网提供的 bookinfo 例子，流量从 productpage 服务进来，productpage 服务会去请求 detail 服务和 reviews 服务，其中 reviews 服务里有3个不同版本的服务会被负载均衡。

## 1) 从 istio 定义的 IngressGateway 进入（例子里表现为 k8s 里的 nodeport）

首先记录 iptbale 的处理包数。

```shell
root@productpage-v1-5ddc866b9-6567x:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    60 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 1 packets, 60 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 743 packets, 68796 bytes)
 pkts bytes target     prot opt in     out     source               destination
   31  1860 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 756 packets, 69576 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    1    60 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   18  1080 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
   13   780 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
   14   840 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001
```
```
root@productpage-v1-5ddc866b9-6567x:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1    60 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 1 packets, 60 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 750 packets, 69388 bytes)
 pkts bytes target     prot opt in     out     source               destination
   34  2040 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 765 packets, 70288 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    1    60 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   19  1140 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
   15   900 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
   16   960 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001
```

从处理包数看出，有3个包进入了 output 链，有一个包来自于 uid 1337 命名空间，直接，有2个包被重定向到了 envoy。

下面用 tcpdump 命令检查一下 tcp 包处理的过程。

```

root@productpage-v1-5ddc866b9-9pz9f:~# tcpdump 'tcp and port 9080'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

23:54:18.716978 IP 172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local.56990 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [P.], seq 1026427780:1026428779, ack 2148586333, win 1323, options [nop,nop,TS val 4591990 ecr 4558159], length 999
23:54:18.717040 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local.56990: Flags [.], ack 999, win 452, options [nop,nop,TS val 4591990 ecr 4591990], length 0
23:54:18.782985 IP productpage-v1-5ddc866b9-9pz9f.43144 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [P.], seq 1133059324:1133060026, ack 2372969541, win 337, options [nop,nop,TS val 4592056 ecr 4184110], length 702
23:54:18.806059 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.43144: Flags [P.], seq 1:338, ack 702, win 380, options [nop,nop,TS val 4218780 ecr 4592056], length 337
23:54:18.806212 IP productpage-v1-5ddc866b9-9pz9f.43144 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [.], ack 338, win 346, options [nop,nop,TS val 4592080 ecr 4218780], length 0
23:54:18.987774 IP productpage-v1-5ddc866b9-9pz9f.51770 > 172-33-22-8.reviews.default.svc.cluster.local.9080: Flags [P.], seq 3368485549:3368486251, ack 1336927343, win 266, options [nop,nop,TS val 4592261 ecr 4458315], length 702
23:54:18.987911 IP 172-33-22-8.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.51770: Flags [.], ack 702, win 282, options [nop,nop,TS val 4592261 ecr 4592261], length 0
23:54:19.378350 IP 172-33-22-8.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.51770: Flags [P.], seq 1:592, ack 702, win 282, options [nop,nop,TS val 4592652 ecr 4592261], length 591
23:54:19.378464 IP productpage-v1-5ddc866b9-9pz9f.51770 > 172-33-22-8.reviews.default.svc.cluster.local.9080: Flags [.], ack 592, win 275, options [nop,nop,TS val 4592652 ecr 4592652], length 0
23:54:19.400369 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local.56990: Flags [P.], seq 1:5893, ack 999, win 452, options [nop,nop,TS val 4592674 ecr 4591990], length 5892
23:54:19.400416 IP 172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local.56990 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 5893, win 1409, options [nop,nop,TS val 4592674 ecr 4592674], length 0
```

可以看出，包从 ip = 172.33.22.5 的 pod 的 56990 端口进入，然后请求到了 productpage pod 的 9080 端口，下一个包，productpage pod 的 43144 端口开始与 detail pod 的 9080 端口开始通信，并且之后在 51770 端口开始与 reviews pod 的 9080 端口进行通信。通讯完了后，与 ip = 172.33.22.5 的 pod 的 56990 端口的通讯返回，并断开连接。

使用 `lsof -i` 命令查看一下端口占用，可以看到，envoy 给每个端口占用都加了名字描述。看到 56990 是envoy 与 `istio-ingressgateway` 这个 pod 通信的一个端口。同理， 43144 端口是 productapage 服务和 details 服务通信的端口，51770 是 productpage 服务与其中一个 reviews pod 通信的端口。

```shell
root@productpage-v1-5ddc866b9-9pz9f:~# lsof -i
COMMAND   PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
pilot-age   1 istio-proxy    5u  IPv4 123790      0t0  TCP productpage-v1-5ddc866b9-9pz9f:45522->istio-pilot.istio-system.svc.cluster.local:15007 (ESTABLISHED)
envoy      11 istio-proxy    9u  IPv4 123811      0t0  TCP localhost:15000 (LISTEN)
envoy      11 istio-proxy   17u  IPv4 123946      0t0  TCP productpage-v1-5ddc866b9-9pz9f:41874->istio-pilot.istio-system.svc.cluster.local:15010 (ESTABLISHED)
envoy      11 istio-proxy   18u  IPv4 123817      0t0  UDP productpage-v1-5ddc866b9-9pz9f:49857->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   19u  IPv4 126574      0t0  TCP productpage-v1-5ddc866b9-9pz9f:15001->172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local:56990 (ESTABLISHED)
envoy      11 istio-proxy   49u  IPv4 123979      0t0  TCP *:15001 (LISTEN)
envoy      11 istio-proxy   50u  IPv4 123980      0t0  UDP productpage-v1-5ddc866b9-9pz9f:55219->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   51u  IPv4 126576      0t0  TCP productpage-v1-5ddc866b9-9pz9f:58182->172-33-102-5.istio-policy.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   52u  IPv4 126624      0t0  TCP productpage-v1-5ddc866b9-9pz9f:53538->172-33-22-6.istio-telemetry.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   53u  IPv4 126627      0t0  TCP productpage-v1-5ddc866b9-9pz9f:50344->zipkin.istio-system.svc.cluster.local:9411 (ESTABLISHED)
envoy      11 istio-proxy   54u  IPv4 126591      0t0  TCP productpage-v1-5ddc866b9-9pz9f:43144->172-33-36-2.details.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   55u  IPv4 126596      0t0  TCP productpage-v1-5ddc866b9-9pz9f:34542->172-33-102-4.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   58u  IPv4 126685      0t0  TCP productpage-v1-5ddc866b9-9pz9f:51770->172-33-22-8.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   59u  IPv4 126751      0t0  TCP productpage-v1-5ddc866b9-9pz9f:42232->172-33-36-7.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
```

## 2) 从 其他 pod 容器 curl service 的名字的流量

因为 k8s 会帮 pod 做服务名的 dns 解析，我们要请求 productpage，就要进入其他的服务的 pod 里进行 curl。这里用 `ratings` pod。

执行：

```shell
$ kubectl get pods | grep ratings | awk '{print($1)};' | xargs -I {} kubectl exec -i {} curl productpage:9080/productpage

... (curl 的结果)
```

查看一下 iptables 处理的包和 tcpdump 的结果：

```shell
root@productpage-v1-5ddc866b9-9pz9f:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    2   120 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 2 packets, 120 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 2549 packets, 234K bytes)
 pkts bytes target     prot opt in     out     source               destination
  109  6540 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 2615 packets, 238K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   43  2580 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
   66  3960 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
   68  4080 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001

```

```shell
root@productpage-v1-5ddc866b9-9pz9f:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    2   120 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 2 packets, 120 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 2554 packets, 234K bytes)
 pkts bytes target     prot opt in     out     source               destination
  112  6720 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 2622 packets, 238K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    2   120 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   44  2640 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
   68  4080 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
   70  4200 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001

````

```shell
root@productpage-v1-5ddc866b9-9pz9f:~# tcpdump  'tcp and (port 9080)'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes


01:31:41.893128 IP 172-33-22-3.ratings.default.svc.cluster.local.40248 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [P.], seq 1559746929:1559747549, ack 632117358, win 812, options [nop,nop,TS val 10435164 ecr 10415676], length 620
01:31:41.893219 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172-33-22-3.ratings.default.svc.cluster.local.40248: Flags [.], ack 620, win 304, options [nop,nop,TS val 10435164 ecr 10435164], length 0
01:31:41.948631 IP productpage-v1-5ddc866b9-9pz9f.43144 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [P.], seq 1133078278:1133078980, ack 2372978620, win 589, options [nop,nop,TS val 10435222 ecr 10050441], length 702
01:31:41.949095 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.43144: Flags [.], ack 702, win 676, options [nop,nop,TS val 10070009 ecr 10435222], length 0
01:31:41.956099 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.43144: Flags [P.], seq 1:337, ack 702, win 676, options [nop,nop,TS val 10070016 ecr 10435222], length 336
01:31:41.956223 IP productpage-v1-5ddc866b9-9pz9f.43144 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [.], ack 337, win 597, options [nop,nop,TS val 10435230 ecr 10070016], length 0
01:31:41.971670 IP productpage-v1-5ddc866b9-9pz9f.51770 > 172-33-22-8.reviews.default.svc.cluster.local.9080: Flags [P.], seq 3368491867:3368492569, ack 1336932662, win 349, options [nop,nop,TS val 10435245 ecr 10380601], length 702
01:31:41.971739 IP 172-33-22-8.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.51770: Flags [.], ack 702, win 380, options [nop,nop,TS val 10435245 ecr 10435245], length 0
01:31:42.177826 IP 172-33-22-8.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.51770: Flags [P.], seq 1:592, ack 702, win 380, options [nop,nop,TS val 10435451 ecr 10435245], length 591
01:31:42.177858 IP productpage-v1-5ddc866b9-9pz9f.51770 > 172-33-22-8.reviews.default.svc.cluster.local.9080: Flags [.], ack 592, win 358, options [nop,nop,TS val 10435451 ecr 10435451], length 0
01:31:42.186483 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172-33-22-3.ratings.default.svc.cluster.local.40248: Flags [P.], seq 1:5893, ack 620, win 304, options [nop,nop,TS val 10435460 ecr 10435164], length 5892
01:31:42.186527 IP 172-33-22-3.ratings.default.svc.cluster.local.40248 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 5893, win 904, options [nop,nop,TS val 10435460 ecr 10435460], length 0


root@productpage-v1-5ddc866b9-9pz9f:~# lsof -i

COMMAND   PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
pilot-age   1 istio-proxy    5u  IPv4 123790      0t0  TCP productpage-v1-5ddc866b9-9pz9f:45522->istio-pilot.istio-system.svc.cluster.local:15007 (ESTABLISHED)
envoy      11 istio-proxy    9u  IPv4 123811      0t0  TCP localhost:15000 (LISTEN)
envoy      11 istio-proxy   17u  IPv4 123946      0t0  TCP productpage-v1-5ddc866b9-9pz9f:41874->istio-pilot.istio-system.svc.cluster.local:15010 (ESTABLISHED)
envoy      11 istio-proxy   18u  IPv4 123817      0t0  UDP productpage-v1-5ddc866b9-9pz9f:49857->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   19u  IPv4 126574      0t0  TCP productpage-v1-5ddc866b9-9pz9f:15001->172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local:56990 (ESTABLISHED)
envoy      11 istio-proxy   49u  IPv4 123979      0t0  TCP *:15001 (LISTEN)
envoy      11 istio-proxy   50u  IPv4 123980      0t0  UDP productpage-v1-5ddc866b9-9pz9f:55219->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   51u  IPv4 126576      0t0  TCP productpage-v1-5ddc866b9-9pz9f:58182->172-33-102-5.istio-policy.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   52u  IPv4 126624      0t0  TCP productpage-v1-5ddc866b9-9pz9f:53538->172-33-22-6.istio-telemetry.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   53u  IPv4 126627      0t0  TCP productpage-v1-5ddc866b9-9pz9f:50344->zipkin.istio-system.svc.cluster.local:9411 (ESTABLISHED)
envoy      11 istio-proxy   54u  IPv4 126591      0t0  TCP productpage-v1-5ddc866b9-9pz9f:43144->172-33-36-2.details.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   55u  IPv4 126596      0t0  TCP productpage-v1-5ddc866b9-9pz9f:34542->172-33-102-4.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   56u  IPv4 260502      0t0  TCP productpage-v1-5ddc866b9-9pz9f:15001->172-33-22-3.ratings.default.svc.cluster.local:40248 (ESTABLISHED)
envoy      11 istio-proxy   57u  IPv4 273577      0t0  UDP productpage-v1-5ddc866b9-9pz9f:49378->kube-dns.kube-system.svc.cluster.local:53
envoy      11 istio-proxy   58u  IPv4 126685      0t0  TCP productpage-v1-5ddc866b9-9pz9f:51770->172-33-22-8.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   59u  IPv4 126751      0t0  TCP productpage-v1-5ddc866b9-9pz9f:42232->172-33-36-7.reviews.default.svc.cluster.local:9080 (ESTABLISHED)

```

iptables 处理的结果与 1 一致，从 lsof -i 看出，ratings pod 的 40248 端口与 productpage pod 的 15001 端口通信。

## 3) 从其他 pod 容器 curl service 的 IP 进入的流量

```shell
$ productpage=`kubectl get svc | grep product | awk '{print($1);}'`
$ ip=`kubectl get svc productpage -o jsonpath='{.spec.clusterIP}'`
$ ratings=`kubectl get pods | grep ratings | awk '{print($1);}'`
$ kubectl exec -it ${ratings} curl http://${ip}:9080/productpage
```
```
root@productpage-v1-5ddc866b9-9pz9f:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 1 packets, 84 bytes)
 pkts bytes target     prot opt in     out     source               destination
    3   180 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 4 packets, 264 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 3834 packets, 352K bytes)
 pkts bytes target     prot opt in     out     source               destination
  158  9480 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 3931 packets, 358K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    3   180 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   61  3660 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
   97  5820 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
  100  6000 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001
root@productpage-v1-5ddc866b9-9pz9f:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 1 packets, 84 bytes)
 pkts bytes target     prot opt in     out     source               destination
    3   180 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 4 packets, 264 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 3839 packets, 353K bytes)
 pkts bytes target     prot opt in     out     source               destination
  161  9660 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 3938 packets, 359K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    3   180 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   62  3720 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
   99  5940 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
  102  6120 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001

```
iptables 包处理还是一样的结果。

来看 tcpdump 和 lsof -i

```
root@productpage-v1-5ddc866b9-9pz9f:~# tcpdump  'tcp and (port 9080)'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

02:19:50.989961 IP 172-33-22-3.ratings.default.svc.cluster.local.40248 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [P.], seq 623:1246, ack 5889, win 1343, options [nop,nop,TS val 13324261 ecr 13319254], length 623 02:19:50.989988 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172-33-22-3.ratings.default.svc.cluster.local.40248: Flags [.], ack 1246, win 363, options [nop,nop,TS val 13324261 ecr 13324261], length 0
02:19:50.999811 IP productpage-v1-5ddc866b9-9pz9f.41600 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [P.], seq 702:1404, ack 337, win 245, options [nop,nop,TS val 13324271 ecr 12952215], length 702
02:19:51.003868 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.41600: Flags [P.], seq 337:673, ack 1404, win 260, options [nop,nop,TS val 12957490 ecr 13324271], length 336
02:19:51.003937 IP productpage-v1-5ddc866b9-9pz9f.41600 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [.], ack 673, win 254, options [nop,nop,TS val 13324277 ecr 12957490], length 0
02:19:51.017007 IP productpage-v1-5ddc866b9-9pz9f.34542 > 172-33-102-4.reviews.default.svc.cluster.local.9080: Flags [P.], seq 3126011300:3126012002, ack 3046400669, win 363, options [nop,nop,TS val 13324289 ecr 11230057], length 702
02:19:51.031450 IP 172-33-102-4.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.34542: Flags [P.], seq 1:506, ack 702, win 413, options [nop,nop,TS val 12594799 ecr 13324289], length 505
02:19:51.031510 IP productpage-v1-5ddc866b9-9pz9f.34542 > 172-33-102-4.reviews.default.svc.cluster.local.9080: Flags [.], ack 506, win 371, options [nop,nop,TS val 13324305 ecr 12594799], length 0
02:19:51.042185 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172-33-22-3.ratings.default.svc.cluster.local.40248: Flags [P.], seq 5889:10472, ack 1246, win 363, options [nop,nop,TS val 13324316 ecr 13324261], length 4583
02:19:51.042225 IP 172-33-22-3.ratings.default.svc.cluster.local.40248 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 10472, win 1414, options [nop,nop,TS val 13324316 ecr 13324316], length 0

root@productpage-v1-5ddc866b9-9pz9f:~# lsof -i
COMMAND   PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
pilot-age   1 istio-proxy    5u  IPv4 123790      0t0  TCP productpage-v1-5ddc866b9-9pz9f:45522->istio-pilot.istio-system.svc.cluster.local:15007 (ESTABLISHED)
envoy      11 istio-proxy    9u  IPv4 123811      0t0  TCP localhost:15000 (LISTEN)
envoy      11 istio-proxy   17u  IPv4 123946      0t0  TCP productpage-v1-5ddc866b9-9pz9f:41874->istio-pilot.istio-system.svc.cluster.local:15010 (ESTABLISHED)
envoy      11 istio-proxy   18u  IPv4 123817      0t0  UDP productpage-v1-5ddc866b9-9pz9f:49857->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   19u  IPv4 126574      0t0  TCP productpage-v1-5ddc866b9-9pz9f:15001->172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local:56990 (ESTABLISHED)
envoy      11 istio-proxy   49u  IPv4 123979      0t0  TCP *:15001 (LISTEN)
envoy      11 istio-proxy   50u  IPv4 123980      0t0  UDP productpage-v1-5ddc866b9-9pz9f:55219->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   51u  IPv4 126576      0t0  TCP productpage-v1-5ddc866b9-9pz9f:58182->172-33-102-5.istio-policy.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   52u  IPv4 126624      0t0  TCP productpage-v1-5ddc866b9-9pz9f:53538->172-33-22-6.istio-telemetry.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   53u  IPv4 330860      0t0  TCP productpage-v1-5ddc866b9-9pz9f:48810->zipkin.istio-system.svc.cluster.local:9411 (ESTABLISHED)
envoy      11 istio-proxy   55u  IPv4 126596      0t0  TCP productpage-v1-5ddc866b9-9pz9f:34542->172-33-102-4.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   56u  IPv4 260502      0t0  TCP productpage-v1-5ddc866b9-9pz9f:15001->172-33-22-3.ratings.default.svc.cluster.local:40248 (ESTABLISHED)
envoy      11 istio-proxy   57u  IPv4 330805      0t0  TCP productpage-v1-5ddc866b9-9pz9f:41600->172-33-36-2.details.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   58u  IPv4 126685      0t0  TCP productpage-v1-5ddc866b9-9pz9f:51770->172-33-22-8.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   60u  IPv4 341526      0t0  TCP productpage-v1-5ddc866b9-9pz9f:43926->172-33-36-7.reviews.default.svc.cluster.local:9080 (ESTABLISHED)

```

结果跟用 service 名访问是一致的。

## 4) 从其他 pod 容器的 sidecar 容器 curl service 的名字的流量

```shell
$ kubectl exec -it ratings-v1-7b9fb5464b-pfq7r bash -c istio-proxy
$ curl http://productpage:9080/productpage

```

```
root@productpage-v1-5ddc866b9-9pz9f:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 1 packets, 84 bytes)
 pkts bytes target     prot opt in     out     source               destination
    4   240 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 5 packets, 324 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 4184 packets, 384K bytes)
 pkts bytes target     prot opt in     out     source               destination
  168 10080 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 4289 packets, 391K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    4   240 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   63  3780 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
  105  6300 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
  109  6540 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001

root@productpage-v1-5ddc866b9-9pz9f:~# iptables -nL -t nat -v
Chain PREROUTING (policy ACCEPT 1 packets, 84 bytes)
 pkts bytes target     prot opt in     out     source               destination
    5   300 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain INPUT (policy ACCEPT 6 packets, 384 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 4189 packets, 385K bytes)
 pkts bytes target     prot opt in     out     source               destination
  171 10260 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain POSTROUTING (policy ACCEPT 4296 packets, 391K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain ISTIO_INBOUND (1 references)
 pkts bytes target     prot opt in     out     source               destination
    5   300 ISTIO_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:9080

Chain ISTIO_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ISTIO_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1
   64  3840 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
  107  6420 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0

Chain ISTIO_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination
  112  6720 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001
```

这里看到有变化，开始有流量经过 PREROUTING 链了。有一个包从 PREROUTING 进入，并且被路由到 ISTIO_REDIRECT 链，然后从 OUTPUT 链出来。

发现有不一样，赶紧跟踪一下 tcp 包处理过程。

```
root@productpage-v1-5ddc866b9-9pz9f:~# tcpdump  'tcp and (port 9080)'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

02:46:33.655102 IP 172.33.22.1.50752 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [S], seq 37579681, win 29200, options [mss 1460,sackOK,TS val 14926928 ecr 0,nop,wscale 7], length 0
02:46:33.655147 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.33.22.1.50752: Flags [S.], seq 741248176, ack 37579682, win 28960, options [mss 1460,sackOK,TS val 14926928 ecr 14926928,nop,wscale 7], length 0
02:46:33.655180 IP 172.33.22.1.50752 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 1, win 229, options [nop,nop,TS val 14926928 ecr 14926928], length 0
02:46:33.655243 IP 172.33.22.1.50752 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [P.], seq 1:92, ack 1, win 229, options [nop,nop,TS val 14926928 ecr 14926928], length 91
02:46:33.655253 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.33.22.1.50752: Flags [.], ack 92, win 227, options [nop,nop,TS val 14926928 ecr 14926928], length 0
02:46:33.661838 IP productpage-v1-5ddc866b9-9pz9f.41600 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [P.], seq 702:1404, ack 337, win 287, options [nop,nop,TS val 14926935 ecr 14554092], length 702
02:46:33.665209 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.41600: Flags [P.], seq 337:495, ack 1404, win 314, options [nop,nop,TS val 14559278 ecr 14926935], length 158
02:46:33.665279 IP productpage-v1-5ddc866b9-9pz9f.41600 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [.], ack 495, win 296, options [nop,nop,TS val 14926939 ecr 14559278], length 0
02:46:33.666351 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.41600: Flags [P.], seq 495:673, ack 1404, win 314, options [nop,nop,TS val 14559279 ecr 14926939], length 178
02:46:33.666406 IP productpage-v1-5ddc866b9-9pz9f.41600 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [.], ack 673, win 304, options [nop,nop,TS val 14926940 ecr 14559279], length 0
02:46:33.678559 IP productpage-v1-5ddc866b9-9pz9f.43926 > 172-33-36-7.reviews.default.svc.cluster.local.9080: Flags [P.], seq 1233715461:1233716163, ack 3923406121, win 247, options [nop,nop,TS val 14926951 ecr 13763210], length 702
02:46:33.681170 IP 172-33-36-7.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.43926: Flags [.], ack 702, win 260, options [nop,nop,TS val 14559292 ecr 14926951], length 0
02:46:33.790380 IP 172-33-36-7.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.43926: Flags [P.], seq 1:588, ack 702, win 260, options [nop,nop,TS val 14559403 ecr 14926951], length 587
02:46:33.790456 IP productpage-v1-5ddc866b9-9pz9f.43926 > 172-33-36-7.reviews.default.svc.cluster.local.9080: Flags [.], ack 588, win 256, options [nop,nop,TS val 14927064 ecr 14559403], length 0
02:46:33.796841 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.33.22.1.50752: Flags [P.], seq 1:5933, ack 92, win 227, options [nop,nop,TS val 14927070 ecr 14926928], length 5932
02:46:33.796894 IP 172.33.22.1.50752 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 5933, win 321, options [nop,nop,TS val 14927070 ecr 14927070], length 0
02:46:33.797370 IP 172.33.22.1.50752 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [F.], seq 92, ack 5933, win 321, options [nop,nop,TS val 14927071 ecr 14927070], length 0
02:46:33.800799 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.33.22.1.50752: Flags [F.], seq 5933, ack 93, win 227, options [nop,nop,TS val 14927074 ecr 14927071], length 0
02:46:33.800863 IP 172.33.22.1.50752 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 5934, win 321, options [nop,nop,TS val 14927074 ecr 14927074], length 0


root@productpage-v1-5ddc866b9-9pz9f:~# lsof -i
COMMAND   PID        USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
pilot-age   1 istio-proxy    5u  IPv4 123790      0t0  TCP productpage-v1-5ddc866b9-9pz9f:45522->istio-pilot.istio-system.svc.cluster.local:15007 (ESTABLISHED)
envoy      11 istio-proxy    9u  IPv4 123811      0t0  TCP localhost:15000 (LISTEN)
envoy      11 istio-proxy   17u  IPv4 123946      0t0  TCP productpage-v1-5ddc866b9-9pz9f:41874->istio-pilot.istio-system.svc.cluster.local:15010 (ESTABLISHED)
envoy      11 istio-proxy   18u  IPv4 123817      0t0  UDP productpage-v1-5ddc866b9-9pz9f:49857->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   19u  IPv4 126574      0t0  TCP productpage-v1-5ddc866b9-9pz9f:15001->172-33-22-5.istio-ingressgateway.istio-system.svc.cluster.local:56990 (ESTABLISHED)
envoy      11 istio-proxy   49u  IPv4 123979      0t0  TCP *:15001 (LISTEN)
envoy      11 istio-proxy   50u  IPv4 123980      0t0  UDP productpage-v1-5ddc866b9-9pz9f:55219->istio-statsd-prom-bridge.istio-system.svc.cluster.local:9125
envoy      11 istio-proxy   51u  IPv4 126576      0t0  TCP productpage-v1-5ddc866b9-9pz9f:58182->172-33-102-5.istio-policy.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   52u  IPv4 126624      0t0  TCP productpage-v1-5ddc866b9-9pz9f:53538->172-33-22-6.istio-telemetry.istio-system.svc.cluster.local:9091 (ESTABLISHED)
envoy      11 istio-proxy   53u  IPv4 330860      0t0  TCP productpage-v1-5ddc866b9-9pz9f:48810->zipkin.istio-system.svc.cluster.local:9411 (ESTABLISHED)
envoy      11 istio-proxy   55u  IPv4 126596      0t0  TCP productpage-v1-5ddc866b9-9pz9f:34542->172-33-102-4.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   56u  IPv4 260502      0t0  TCP productpage-v1-5ddc866b9-9pz9f:15001->172-33-22-3.ratings.default.svc.cluster.local:40248 (ESTABLISHED)
envoy      11 istio-proxy   57u  IPv4 330805      0t0  TCP productpage-v1-5ddc866b9-9pz9f:41600->172-33-36-2.details.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   58u  IPv4 126685      0t0  TCP productpage-v1-5ddc866b9-9pz9f:51770->172-33-22-8.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
envoy      11 istio-proxy   60u  IPv4 341526      0t0  TCP productpage-v1-5ddc866b9-9pz9f:43926->172-33-36-7.reviews.default.svc.cluster.local:9080 (ESTABLISHED)
```

发现并没有 50752 的端口建立了连接，再请求一遍，发现这个端口变成了 50756。看来这个端口是临时建立的，请求完了就释放掉了。那么，我们还可以看看这个 172.33.22.1 的 ip 是属于谁的。

由于测试集群是用的 flannel 的 k8s 网络方案，flannel 通过连接不同 node 的 docker0 网桥实现不同 node 上的 docker 不同网段仍能正常访问。所以，pod 上的 ip 是 172.33.22.2，那 172.33.22.1 有可能是 docker 网桥的 ip 地址。

登录 productpage 所在的 node 执行 `ifonnfig` 发现果然如此。

所以，在 istio-proxy sidecar 容器 curl 请求 productpage，流量是从 productpage 所在的 node 的 docker 网桥进来的。

而流量从 docker0 网桥进来的原因是，我刚好选了一个与 productpage pod 同一个 node 的 pod 来实验，换一个不同 node 的 pod 情况如下：
```
03:48:53.486108 IP 172.17.8.102.44264 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [S], seq 2104800333, win 29200, options [mss 1460,sackOK,TS val 18313233 ecr 0,nop,wscale 7], length 0
03:48:53.486259 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.17.8.102.44264: Flags [S.], seq 2430551524, ack 2104800334, win 28960, options [mss 1460,sackOK,TS val 18666760 ecr 18313233,nop,wscale 7], length 0
03:48:53.486658 IP 172.17.8.102.44264 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 1, win 229, options [nop,nop,TS val 18313234 ecr 18666760], length 0
03:48:53.487271 IP 172.17.8.102.44264 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [P.], seq 1:92, ack 1, win 229, options [nop,nop,TS val 18313234 ecr 18666760], length 91
03:48:53.487327 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.17.8.102.44264: Flags [.], ack 92, win 227, options [nop,nop,TS val 18666761 ecr 18313234], length 0
03:48:53.496973 IP productpage-v1-5ddc866b9-9pz9f.41600 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [P.], seq 6318:7020, ack 3032, win 505, options [nop,nop,TS val 18666766 ecr 18202275], length 702
03:48:53.537366 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.41600: Flags [.], ack 7020, win 589, options [nop,nop,TS val 18313285 ecr 18666766], length 0
03:48:53.943255 IP 172-33-36-2.details.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.41600: Flags [P.], seq 3032:3370, ack 7020, win 589, options [nop,nop,TS val 18313690 ecr 18666766], length 338
03:48:53.943338 IP productpage-v1-5ddc866b9-9pz9f.41600 > 172-33-36-2.details.default.svc.cluster.local.9080: Flags [.], ack 3370, win 513, options [nop,nop,TS val 18667217 ecr 18313690], length 0
03:48:53.962742 IP productpage-v1-5ddc866b9-9pz9f.34542 > 172-33-102-4.reviews.default.svc.cluster.local.9080: Flags [P.], seq 1404:2106, ack 1012, win 446, options [nop,nop,TS val 18667236 ecr 17740045], length 702
03:48:53.981229 IP 172-33-102-4.reviews.default.svc.cluster.local.9080 > productpage-v1-5ddc866b9-9pz9f.34542: Flags [P.], seq 1012:1518, ack 2106, win 523, options [nop,nop,TS val 17938348 ecr 18667236], length 506
03:48:53.981336 IP productpage-v1-5ddc866b9-9pz9f.34542 > 172-33-102-4.reviews.default.svc.cluster.local.9080: Flags [.], ack 1518, win 455, options [nop,nop,TS val 18667255 ecr 17938348], length 0
03:48:53.987769 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.17.8.102.44264: Flags [P.], seq 1:4629, ack 92, win 227, options [nop,nop,TS val 18667261 ecr 18313234], length 4628
03:48:53.988322 IP 172.17.8.102.44264 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 4629, win 301, options [nop,nop,TS val 18313735 ecr 18667261], length 0
03:48:53.992174 IP 172.17.8.102.44264 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [F.], seq 92, ack 4629, win 301, options [nop,nop,TS val 18313739 ecr 18667261], length 0
03:48:53.992662 IP productpage-v1-5ddc866b9-9pz9f.9080 > 172.17.8.102.44264: Flags [F.], seq 4629, ack 93, win 227, options [nop,nop,TS val 18667266 ecr 18313739], length 0
03:48:53.994325 IP 172.17.8.102.44264 > productpage-v1-5ddc866b9-9pz9f.9080: Flags [.], ack 4630, win 301, options [nop,nop,TS val 18313740 ecr 18667266], length 0

```
流量从 node 的 ip 和某个端口与 productpage 的 9080 端口通信，而且这个 node 端口也是请求完即销毁的，可见，当同 node 的 pod 流量进入时，请求直接经过 docker0 网桥转发即可，而不同 node 请求时，productpage pod 会看到请求是以 node 为请求源发起请求，这个请求是由 flannel 转发的。

那么还有一个问题，为什么由 istio-proxy curl 的流量会经过 docker0 网桥，并经过 iptables PREROUTING 链，而在 details，或者 ratings pod curl 的请求不会经过呢？

通过 `/proc/net/nf_conntrack` 文件可以看到包被处理的过程：

```shell
root@productpage-v1-b48b94974-hp9x4:~# cat /proc/net/nf_conntrack | grep 9080
ipv4     2 tcp      6 61 TIME_WAIT src=172.33.102.10 dst=10.254.224.87 sport=54598 dport=9080 src=127.0.0.1 dst=172.33.102.10 sport=15001 dport=54598 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
ipv4     2 tcp      6 61 TIME_WAIT src=127.0.0.1 dst=127.0.0.1 sport=35156 dport=9080 src=127.0.0.1 dst=127.0.0.1 sport=9080 dport=35156 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
ipv4     2 tcp      6 60 TIME_WAIT src=172.33.102.10 dst=10.254.96.234 sport=45460 dport=9080 src=127.0.0.1 dst=172.33.102.10 sport=15001 dport=45460 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
ipv4     2 tcp      6 61 TIME_WAIT src=172.17.8.102 dst=172.33.102.10 sport=49770 dport=9080 src=172.33.102.10 dst=172.17.8.102 sport=15001 dport=49770 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
```

这个是在 istio-proxy sidecar 容器请求的包处理结果，而下面是在 ratings 容器里请求的包处理结果。

```shell
root@productpage-v1-b48b94974-hp9x4:~# cat /proc/net/nf_conntrack | grep 9080
ipv4     2 tcp      6 116 TIME_WAIT src=172.33.102.10 dst=10.254.96.234 sport=45632 dport=9080 src=127.0.0.1 dst=172.33.102.10 sport=15001 dport=45632 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
ipv4     2 tcp      6 116 TIME_WAIT src=127.0.0.1 dst=127.0.0.1 sport=35328 dport=9080 src=127.0.0.1 dst=127.0.0.1 sport=9080 dport=35328 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
ipv4     2 tcp      6 116 TIME_WAIT src=172.33.102.10 dst=10.254.224.87 sport=54770 dport=9080 src=127.0.0.1 dst=172.33.102.10 sport=15001 dport=54770 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
```

在 istio-proxy 容器（ratings 的 istio-proxy sidecar 容器）里请求的，被请求的服务（productpage 服务的pod）多了一个从 istio-proxy 容器来自的 node 的包（ratings pod 部署在 node2，ip为172.17.8.102）

这种差异可以在 ratings istio-proxy sidecar 里的请求处理情况推断出来。

```
# 在 ratings 生产容器里的请求
root@ratings-v1-6bdbc7b978-s5dmx:~# cat /proc/net/nf_conntrack | grep 10.254.131.126
ipv4     2 tcp      6 117 TIME_WAIT src=172.33.36.2 dst=10.254.131.126 sport=49912 dport=9080 src=127.0.0.1 dst=172.33.36.2 sport=15001 dport=49912 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
root@ratings-v1-6bdbc7b978-s5dmx:~#
root@ratings-v1-6bdbc7b978-s5dmx:~# cat /proc/net/nf_conntrack | grep 10.254.131.126
# 在 ratings istio-proxy sidecar 容器里的请求后，多了一条
ipv4     2 tcp      6 113 TIME_WAIT src=172.33.36.2 dst=10.254.131.126 sport=49920 dport=9080 src=10.254.131.126 dst=172.33.36.2 sport=9080 dport=49920 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
ipv4     2 tcp      6 76 TIME_WAIT src=172.33.36.2 dst=10.254.131.126 sport=49912 dport=9080 src=127.0.0.1 dst=172.33.36.2 sport=15001 dport=49912 [ASSURED] mark=0 secctx=system_u:object_r:unlabeled_t:s0 zone=0 use=2
```

可以看到，生产容器里的请求其实是由重写到 envoy 并与端口通信的，而 sidecar 容器里的请求是真真实实与 `10.254.131.126` 通信的，这其实也验证了 istio 对 pod 的两条 iptables 规则：
```
root@productpage-v1-5ddc866b9-6567x:~# iptables -nL -t nat -v
...
Chain ISTIO_OUTPUT (1 references)
... 
 pkts bytes target     prot opt in     out     source               destination
    3   180 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
    0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
... 
    0     0 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0
... 
```

来自用户空间 1337 的请求，直接返回，被直接返回的请求则经过 docker0 网桥，再经过 flannel 网桥，进入 prodcutpage 所在的 node 的 docker0 网桥，于是会触发 iptables 的 PREROUTING 链。

而其他请求，则会重定向有 envoy 处理，而 envoy 已经创建了与各个 pod 的长连接。当流量经过 envoy 请求 productpage 时，请求直接由长连接处理了，所以 iptables PREROUTING 链处理的包数不是没变，而是在长连接建立时变化过（这个可以在 bookinfo 刚部署的时候看到），之后就不再变化了。


## 5) 从 其他 pod 容器的 sidecar 容器 curl service 的 IP 进入的流量

从 /proc/net/nf_conntrack 文件看到，实际上 service name 在 k8s 里是作为 dns 名的，可以推断出请求 service name 和 service ip 的结果是一样的。

## 6) 从 productpage pod curl localhost 进入的流量

可以推断出，请求会经过 lo 口，不会经过 PREROUTING 链，然后在请求时，会有3个请求（ratings reviews details）经过 OUTPUT 链，请求 productpage 服务的会被直接返回，而其他2个会被重定向到 ISTIO_REDIRET 链处理。实验后，数据也符合推断。

## 7) 从 productpage istio-proxy sidecar 容器 curl localhost 进入的流量

与 6）是差不多的，唯一不一样的地方是，请求在经过 ISTIO_OUTBOUND 链时，6）经过的是这条规则：
```
    3   180 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
```
而 7）经过的是这条：
```
    0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
```

# 4. 总结

这篇文章在宋大的文章基础上做了一些本地环境的实践，通过 `iptables`、`tcpdump` 命令和查看 `/proc/net/nf_conntrack` 文件的方式，跟踪了 istio 对 pod 流量的劫持以及劫持过程请求和数据包的处理，验证了 istio 的设计。在实验过后，对流量劫持的过程更加熟悉了。

参考资料:

- [理解 Istio Service Mesh 中 Envoy 代理 Sidecar 注入及流量劫持](https://jimmysong.io/posts/envoy-sidecar-injection-in-istio-service-mesh-deep-dive/)
- [iptables 工作原理](https://wangchujiang.com/linux-command/c/iptables.html#%E5%B7%A5%E4%BD%9C%E6%9C%BA%E5%88%B6)
- [浅析 /proc/net/nf_conntrack 文件（转）](http://blog.51cto.com/9346709/2328839)
- [DockOne微信分享（一七五）：盘点Kubernetes网络问题的4种解决方案](http://dockone.io/article/5992)
