---
author: "trainyao"
title: golang 使用方法 & 备忘
linktitle: git 使用方法 & 备忘
categories: ["devops"]
weight: 10
---

# tool chain

## go vet

`go vet` 是 golang 官方自带的代码静态检查工具, 可以帮助程序员检查出编译不会出错但有 bug 的代码

### 用法

go vet [./path/to/package `相对路径` | path/to/`gopath 能访问到的 package` | path/to/package 的`绝对路径`]

### 分析器

go vet 自带多个分析器, 遍历入参的目录/文件, 执行每个分析器, 找出代码的问题

分析器有(可以通过 go tool vet help 看到):
- 语义错误
    - `bools`, ex: `if a == 1 && a == 0` `if a == 1 && a == 1`
    - `ifaceassert`, ex: 
      ```go
      var itf interface{
        Read()
      }
      newPtr := itf.(io.Reader)
      ```

## golangci-lint

# unsafe 包

## unsafe.SizeOf
