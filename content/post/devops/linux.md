---
author: "trainyao"
title: linux 使用方法 & 备忘
linktitle: linux 使用方法 & 备忘
categories: ["devops"]
weight: 10
---

- lsb_release -cs 查看 linux 发行号


- usermod -a -G [group] [username] / groupmems -g [group] -a [username]
- 修改大文件
    - tail -n +3 old > new, -n 参数支持+号
- nat
    - dnat
        - 入
        - lb 对请求包进行目标 IP 转换
    - snat
        - 出
        - lb 对目标的回包, 进行来源 IP 转换, 改为 lb 的 IP, 返回给请求方

        fsfsdfadfa
