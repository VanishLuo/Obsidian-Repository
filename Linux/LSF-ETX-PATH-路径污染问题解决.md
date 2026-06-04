---
tags: [问题解决]
---

# LSF 集群 ETX 共享桌面 PATH 路径污染问题解决

## 问题描述

在 LSF 集群环境中使用 ETX 共享桌面时，用户在某个登录节点使用很久之后，PATH 路径被污染，导致新打开的终端使用的是旧的 PATH 路径。

## 解决方案

在 `.bashrc` 文件中先初始化 PATH 路径，然后再加载所需要的配置。

```bash
# 初始化 PATH
export PATH=/usr/local/bin:/usr/bin:/bin

# 后续加载其他配置
source /path/to/your/config
```

## 说明

通过在 `.bashrc` 开头重新初始化 PATH，可以确保每次打开新终端时都使用正确的路径，避免旧 PATH 的累积污染。
