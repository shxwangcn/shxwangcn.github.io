---
title: 聊聊golang中的slice
date: 2022-04-12 23:21:33 +0800
categories: [golang, types & containers, slice]
tags: golang slice
mermaid: true
---

本篇文章聊聊 golang 中的核心数据结构 —— slice 的内部实现，从而保证我们能够正确且高效地使用它。

源码文件：

- slice实现：[runtime/slice.go](https://github.com/golang/go/blob/master/src/runtime/slice.go)
- append函数: [cmd/compile/internal/walk/builtin.go](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/builtin.go)