---
uid: gdb-debug-urg1
title: GDB 调试 Synopsys urg1
tags:
  - GDB
  - Synopsys
  - urg1
  -调试
created: 2026-06-05
description: 使用 GDB 调试 Synopsys urg1 软件，含 --args 参数传递方法
---

## 背景

urg1 是 Synopsys 提供的覆盖率报告生成工具，运行时常需要带参数。

GDB 调试带参数的可执行文件，需要用 `--args` 将参数传递给被调试程序。

## 基本用法

### 启动调试

```bash
gdb --args ./urg1 -64 -show-races -write-info info.races /path/to/your/design
```

或者进入 gdb 交互界面后再设置：

```bash
$ gdb ./urg1
(gdb) set args -64 -show-races -write-info info.races /path/to/your/design
(gdb) run
```

### 常用命令速查

| 命令 | 缩写 | 说明 |
| --- | --- | --- |
| `run` | `r` | 开始运行 |
| `backtrace` | `bt` | 查看调用栈 |
| `frame N` | `f N` | 切换到第 N 帧 |
| `disassemble` | `disas` | 反汇编当前函数 |
| `break location` | `b` | 设置断点 |
| `info breakpoints` | `i b` | 查看断点列表 |
| `delete breakpoints N` | `d N` | 删除第 N 个断点 |
| `continue` | `c` | 继续运行 |
| `next` | `n` | 单步执行（不进入函数） |
| `step` | `s` | 单步执行（进入函数） |
| `print var` | `p var` | 打印变量值 |
| `x/nfu addr` | | 查看内存 |
| `quit` | `q` | 退出 GDB |

### 示例：查看崩溃时的调用栈

```bash
$ gdb --args ./urg1 -64 -show-races -write-info info.races /path/to/design
GNU gdb (GDB) ...
Reading symbols from ./urg1...(no debugging symbols found)...done.
(gdb) run
# 等待程序崩溃或断点触发

Program received signal SIGSEGV, Segmentation fault.
(gdb) backtrace
#00x00002aaaaacd8418e3 in SNPSle_126d17af453e054a () from .../libvcsnew.so
#10x00002aaaaacd8417cd in SNPSle_2c2186f0f2f42415 () from .../libvcsnew.so
...
(gdb) frame 0
(gdb) disas
```

### 断点设置示例

```bash
# 在特定函数处设断点
(gdb) break SNPSle_126d17af453e054a

# 在特定地址处设断点
(gdb) break *0x2aaaaacd8418e3

# 在特定文件:行号处设断点
(gdb) break some_file.c:123
```

## 动态库找不到的问题（LD_LIBRARY_PATH）

### 问题现象

用户平时直接运行 `urg` 命令（由 VCS 安装包提供的包装脚本），VCS 内部会调用 `linux64/bin/urg1`。

但在 GDB 调试时如果直接用 `urg1` 的绝对路径启动，程序运行时可能会报错：

```bash
error while loading shared libraries: libXXX.so: cannot open shared object file: No such file or directory
```

这是因为 `urg1` 依赖 VCS 自带的动态链接库（如 `libvcsnew.so`、`libsnpsmalloc.so` 等），这些 `.so` 文件位于 VCS 安装目录的 `linux64/lib/` 下，系统默认不会搜索这个路径。

### 解决方案

使用 `LD_LIBRARY_PATH` 环境变量，将 VCS 的 `linux64/lib` 目录加入动态库搜索路径。

### 方法一：在 GDB 启动前设置环境变量

```bash
export LD_LIBRARY_PATH=/path/to/vcs/Q-2020.03/linux64/lib:$LD_LIBRARY_PATH
gdb --args /path/to/vcs/Q-2020.03/linux64/bin/urg1 -64 -show-races -write-info info.races /path/to/design
```

### 方法二：在 GDB 交互界面中设置

```bash
$ gdb /path/to/vcs/Q-2020.03/linux64/bin/urg1
(gdb) set environment LD_LIBRARY_PATH=/path/to/vcs/Q-2020.03/linux64/lib
(gdb) set args -64 -show-races -write-info info.races /path/to/design
(gdb) run
```

### 方法三：直接通过 gdb 的 `set env` 命令一行搞定

```bash
gdb /path/to/vcs/Q-2020.03/linux64/bin/urg1
(gdb) set environment LD_LIBRARY_PATH=/path/to/vcs/Q-2020.03/linux64/lib
(gdb) run -64 -show-races -write-info info.races /path/to/design
```

### 原理说明

`LD_LIBRARY_PATH` 是 Linux 下控制动态链接器（`ld.so`）搜索 `.so` 库文件的路径环境变量，类似于 Windows 的 `PATH`。

当程序调用 `dlopen`、`dlsym` 或直接依赖某个 `.so` 时，动态链接器会按以下顺序搜索：

1. `LD_LIBRARY_PATH` 中的路径（优先）
2. `rpath`（编译时写入可执行文件的路径）
3. `/etc/ld.so.cache`（系统缓存）
4. `/lib` 和 `/usr/lib`（系统默认路径）

通过把 VCS 的 `linux64/lib` 目录加入 `LD_LIBRARY_PATH`，调试器启动 `urg1` 时就能正确找到 Synopsys 私有库，避免 `cannot open shared object file` 错误。

### 验证库是否找到

调试时可在 gdb 中使用 `info sharedlibrary` 查看已加载的共享库及其路径：

```bash
(gdb) info sharedlibrary
```

如果库路径显示为 `(outside)` 或 `No symbols`，说明库加载正常；如果显示 `Failed to read` 则说明库路径不对。
