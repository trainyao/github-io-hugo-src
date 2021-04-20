---
author: "trainyao"
date: 2021-04-20
title: ssh 使用方法 & 备忘
linktitle: ssh 使用方法 & 备忘
categories: ["devops"]
weight: 10
---

- server 端 /var/log/auth.log 可以看日志
- client 端 ssh user@host -pport -vvvv 可以看 client 和 server 端交互过程, 及来回的错误码
- sshd_config 配置项不可覆盖, 详情可以用 -T 调试配置效果
- sshd -T | grep [config] 可以实时看 ssh 配置效果, 有时候配置值没生效可以定位出来
- root 免密登录需要配置 /root/.ssh/authorized_keys 公钥, 以及配置 sshd_config PermitRootLogin yes
    - PermitRootLogin 有 yes, prohibit-password, without-password, forced-commands-only, no 选项, 默认 prohibit-password
- 有博文说 /root 目录下的文件夹和文件的权限和拥有者, 会影响 root ssh 登录, 这个待验证

ref:

- [https://stackoverflow.com/questions/41684706/login-refused-for-root-via-ssh](https://stackoverflow.com/questions/41684706/login-refused-for-root-via-ssh)
- [https://superuser.com/questions/1137438/ssh-key-authentication-fails](https://superuser.com/questions/1137438/ssh-key-authentication-fails)
