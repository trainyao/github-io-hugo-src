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
