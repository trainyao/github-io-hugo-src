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
