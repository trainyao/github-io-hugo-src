---
author: "trainyao"
title: k8s 使用方法 & 备忘
linktitle: k8s 使用方法 & 备忘
categories: ["devops"]
weight: 10
---

-  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
- echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
- curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
- echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
- apt-get install docker-ce docker-ce-cli containerd.io
- apt-get install \
    apt-transport-https \
        ca-certificates \
            curl \
                gnupg \
                    lsb-release


- kubeadm init --kubernetes-version=1.18.1 --apiserver-advertise-address=192.168.128.130 --image-repository registry.aliyuncs.com/google_containers  --service-cidr=10.1.0.0/16  --pod-network-cidr=10.244.0.0/16
    - ref [https://www.cnblogs.com/xiao987334176/p/12696740.html](https://www.cnblogs.com/xiao987334176/p/12696740.html)
- 安装 flannel: wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml && grep 10.244 -n kube-flannel.yml && sed -i 's/10.244.0.0\/16/[pod network cidr]/' && kubectl apply -f kube-flannel.yml

- cni
    - cni 插件工作原理
        - init config to /etc/cni/net.d/*-*.conflist
        - kubelet 内的 CRI 实现, dockershim 加载配置文件
        - kubelet 根据配置文件, 依次调用 CNI 插件, 初始化 pause 容器, 配置容器网络
    - 细节
        - hairpin mode, 配置 cni 网桥 veth 设备接收数据包的功能, 确保容器可以在容器网络里访问自己
    - plugins
        - plugins 都依照 [containernetworking/cni/skel](https://github.com/containernetworking/cni/tree/master/pkg/skel) 做基本框架
        - bridge
            - source code [containernetworking/plugins/bridge](https://github.com/containernetworking/plugins/blob/master/plugins/main/bridge/bridge.go)
			- 依托 lib [github.com/vishvananda/netlink](https://github.com/vishvananda/netlink) 做 ip link add/set 命令操作
        - cilium (v1.9.8)
            - 安装
                - 安装过程颇容易, 官网有提供 operator 直接一键安装 `kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.9/install/kubernetes/quick-install.yaml -n kube-system` [官方 install doc](https://docs.cilium.io/en/v1.9/gettingstarted/k8s-install-default/)
                - 前置依赖
                    - 挂在 bpf 目录 `sudo mount bpffs /sys/fs/bpf -t bpf && echo 'bpffs        /sys/fs/bpf      bpf     defaults 0 0' >> /etc/fstab` 
                    - 打开内核 cilium 依赖的关于 bpf 的参数 via `/boot/config-$(uname -r)` 参考[官方文档描述](https://docs.cilium.io/en/v1.9/operations/system_requirements/#linux-kernel)
                      ```
                      CONFIG_BPF=y
                      CONFIG_BPF_SYSCALL=y
                      CONFIG_NET_CLS_BPF=y
                      CONFIG_BPF_JIT=y
                      CONFIG_NET_CLS_ACT=y
                      CONFIG_NET_SCH_INGRESS=y
                      CONFIG_CRYPTO_SHA1=y
                      CONFIG_CRYPTO_USER_API_HASH=y
                      CONFIG_CGROUPS=y
                      CONFIG_CGROUP_BPF=y
                      ```
            - 调试
                - [官方提供的连接测试用例](https://docs.cilium.io/en/v1.9/gettingstarted/k8s-install-default/#deploy-the-connectivity-test) `kubectl create ns cilium-test && kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.9/examples/kubernetes/connectivity-check/connectivity-check.yaml -n cilium-test`
                - 其他博主调整过的用例 `kubectl apply -n cilium-test -f https://github.com/nevermosby/K8S-CNI-Cilium-Tutorial/raw/master/cilium/connectivity-check.yaml` 参考[博文](https://cilium.io/blog/2020/05/04/guest-blog-kubernetes-cilium)
                - 可以使用不同主机上的 pod 互相访问 测试联通性, `docker run --rm trainyao/toolbox:ubuntu-18-40 -- ping [pod IP from other worker]`
            - 压测
                - 压测数据 (千兆网卡)
                    - iperf 测得 `907Mbyte/s`
                    - siege -c 100 -t 2M
                        - 测得 service 访问 pod 4k qps 左右, 1.79M/s 吞吐量
                        - 测得 headless service 访问 pod 4k qps 左右, 2M/s 吞吐量
                        - 测得不同节点 pod ip qps 4k qps 左右, 3M/s 吞吐量
                - 和 flannel 对比
                    - cilium 换 flannel 过程中还发现 flannel daemonest 会报 `address already used`, 重启节点之后就好了, 怀疑是因为 cilium 配置过宿主机的网卡之类的, 重启之后配置重置过了
                    - iperf 测得 `910Mbyte/s`
                    - siege -c 100 -t 2M
                        - 测得 service 访问 pod 4k qps 左右, 4M/s 吞吐量
                        - 测得 headless service 访问 pod 4k qps 左右, 4M/s 吞吐量
                        - 测得不同节点 pod ip qps 5k qps 左右, 3M/s 吞吐量
                - 怀疑可能是节点规模没上去, iptable / eBPF 差距应该没这么大? 待验证

- kube-proxy (v1.17.17)
    - 主要通过 proxyServer 里的 proxier 来响应 k8s 资源变化, 实现了多种接口: `k8s.io/kubernetes/pkg/proxy/config` 里的 `config.EndpointHandler`, `config.EndpointSlaceHandler`, `config.ServiceHandler`, `config.NodeHandler`, 可以代码查找下列 interface, 主要关注不同 proxier 对上述接口的实现
    - kube-proxy 实现了 3 种 proxier: `iptables`, `ipvs`, `userspace`
    - 从代码来看, ipvs 还分 `ipvs proxier` 和 `ipvs dual stack proxier`, 好像是对 ipv6 的支持
    - 从 interface implement 来看, 还实现了一个 `win userspace proxier`
    - iptables 实现:
        - 通过 `pkg/proxy/iptables/proxier.go`.`syncProxyRules()` 执行 node 变化后 iptables 的同步
        - 操作 iptables 通过 `pkg/util/iptables`.`runner` 实现了 `pkg/util/iptables`.`Interface`
        - `pkg/util/iptables`.`runner` 实际上是执行 `iptables` `ip6tables` 命令, runner 里还有一个实现了 `k8s.io/utils/exec`.`Interface` 的 `k8s.io/util/exec`.`executor`
            - 其中还包括一个 `k8s.io/utils/exec`.`Cmd` 的接口抽象 `k8s.io/utils/exec`.`cmdWrapper`, 它包装了 `os/exec`.`Cmd`

- scheduler 调度器
    - default scheduler
        - 根据 filter 规则筛选出符合条件的 node (Predicates 阶段,算法), 再用 rating 规则给 node 打分 (Priority 阶段,算法), 最高分 node 作为调度结果
            - filter 规则有多种: 如判断 pod 资源 request 和 node 剩余资源是否符合, PV 所在的 node, 自定义 pod 资源, affinity 等等的维度
        - 调度结果是给 pod 的 spec.NodeName 设置 node 的名字
        - kubelet 找到有自己的 pod, 就会去部署?
        - kubelet 在部署 pod 前还会执行一个 admit 阶段(实际上是执行 predicates 阶段的逻辑), 重复检查确认, 判断是否符合 node 调度条件, 以免感知时间差过程中, node 状态发生了变化
        - kubelet admit 后, scheduler 会真正设置 nodename 到 pod 信息中, 这个步骤称为 "bind"
        - 调度速度优化
            - cache 化, 调度前 scheduler 会将 node 的信息读取一份到内存, 避免计算过程需要重复获取 node 信息
            - 无锁化, 调度过程需要出队 scheduler FIFO/Priority Queue, 以及更新 cache, 尽在获取任务和更新 cache 才需要获取锁
            - 根据 node 并行化, 计算后再写到 scheduler cache 中
    - default scheduler 扩展机制
        - go plugins 形式在调度的各个阶段, 提供插件机制
        - 阶段:
            - PreFilter
            - PostFilter
            - PostScoring
            - PreBind
            - PostBind

- service
	- clusterip
		- 通过 kube-proxy 监听 service 和对应 pod 变化, 在宿主机上更改 iptables / ipvs 配置, 实现 service 负载均衡功能
			- kube-proxy 普通模式通过 iptables --probiility 控制 service ip 到 pod 的流量
			- kube-proxy 还支持 --proxy-mode=ipvs (可以在 -n kube-system daemonSet kube-proxy 启动参数指定), 优化了 pod 数量对 iptables 管理的复杂性
	- headless service (clusterip = None)
		- 通过 kube-dns / 其他 dns 实现, 控制 service dns 解析返回的结果, 直接在 dns 记录里返回 pod ip
	- nodeport
		- node 端口直接由 kube-proxy 进程监听
		- iptables, proxy-mode=ipvs 都没有看到对应的规则
		- 疑似是用户态的代理?
		- service spec.exernalTraficPolicy: local, 指定 pod 回包时不用 SNAT 改为源 node 的 IP (client 请求的 node), 而是直接返回给 client
	- loadbalancer
		- cloudprovider 转接层
	- externalname
		- kube-dns 中将 service 域名解析成一个外部 cname
	- 其他
		- spec.externalIPs
			- 访问配置的 ip 可以直接访问到 pod
			- 需要 client 能访问 ip 可以直接访问到至少一个 node
			- 实际上流量会先经过配置的 IP, 再代理到 node

 ref:
 - [张磊大大的 k8s 课程](https://time.geekbang.org/column/intro/116)
 - [k8s 源码](https://github.com/kubernetes/kubernetes.git)
