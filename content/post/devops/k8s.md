---
author: "trainyao"
title: k8s 使用方法 & 备忘
linktitle: k8s 使用方法 & 备忘
categories: ["devops"]
weight: 10
---

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

- kube-proxy (v1.17.17)
    - 主要通过 proxyServer 里的 proxier 来响应 k8s 资源变化, 实现了多种接口: `k8s.io/kubernetes/pkg/proxy/config` 里的 `config.EndpointHandler`, `config.EndpointSlaceHandler`, `config.ServiceHandler`, `config.NodeHandler`, 可以代码查找下列 interface, 主要关注不同 proxier 对上述接口的实现
    - kube-proxy 实现了 3 种 proxier: `iptables`, `ipvs`, `userspace`
    - 从代码来看, ipvs 还分 `ipvs proxier` 和 `ipvs dual stack proxier`, 好像是对 ipv6 的支持
    - 从 interface implement 来看, 还实现了一个 `win userspace proxier`
    - iptables 实现:
        - 

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
