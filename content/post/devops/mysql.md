---
author: "trainyao"
title: mysql 使用方法 & 备忘
linktitle: mysql 使用方法 & 备忘
categories: ["devops"]
weight: 10
---


- set FOREIGN_KEY_CHECKS=0; 可以避免 discard table 外键错误 "cannot delete or update a parent row a foreign key constraint fails"
- 'Constant, random or timezone-dependent expressions in (sub)partitioning function are not allowed' 分区函数嵌套使用, 会出这个错误, 比如 to_days(mode()), 好像是因为 to_days 参数不能是 random 的
