---
title: 总结Linux中常见的命令行工具
date: 2022-04-21 09:20:12 +0800
categories: [linux,commands]
tags: linux,find,grep,sed,awk,netstat,
mermaid: true
---

本篇文章总结下日常开发中比较常用的命令行工具。

> 持续更新中...

包括以下工具：

- 文本文件类：
    - 文件查找 find
    - 文件搜索 grep
    - 文件替换 sed
    - 文件分析 awk
- 网络分析类：
    - netstat
    - nc
    - nslookup & dig
    - tcpdump
- 文件传输&下载：
    - wget
    - curl
    - rsync
- 其它：
    - xargs

本篇文章会不定期更新，以便新增其它的命令。此外，有些命令因为过于复杂，所以会使用单独的文章来记录：

- 使用 iptables 管理防火墙

对于每个命令，这里只会记录我日常用到的功能，以便随时查看。对于每个命令的完整性使用指南，还是 `man xxx` 吧。

# 1 文本文件操作工具

## find

功能：支持通过文件的类型、名称、时间等过滤器来查找文件。支持正则表达式条件。

使用语法：

```sh
find [-H | -L | -P] [-EXdsx] [-f path] path ... [expression]
```

包括三部分：

- options 控制find的行为，其中，`-H | -L | -P` 控制对软连接的处理逻辑；`-Exdsx`等各有用处，具体参考man手册；
- Primaries 控制查找的条件，支持按文件类型、时间、权限等等信息来查找；
- path 查找的目录

下面给出几个示例：

**示例1 找出当前目录下所有的有可执行权限的普通文件**


```sh
find . -type f -perm 
```

## grep

## sed

## awk

# 2 网络分析工具

## netstat

## nc

## nslookup & dig

## tcpdump

# 3 文件传输&下载

## wget

## curl

## rsync
