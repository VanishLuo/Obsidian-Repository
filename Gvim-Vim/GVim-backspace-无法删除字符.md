---
tags: [gvim, vim]
---

# GVim backspace 无法删除字符

## 问题描述

在 GVim 的 Insert 模式下，按 Backspace 键无法删除之前输入的字符。

## 解决方法

在 `~/.vimrc` 或 GVim 的配置文件中添加：

```vim
set backspace=2
```

或使用更完整的配置：

```vim
set backspace=indent,eol,start
```

## 说明

- `indent` - 允许删除缩进
- `eol` - 允许删除换行符
- `start` - 允许删除进入 Insert 模式前输入的字符
- `set backspace=2` 等同于上述三个选项的组合
