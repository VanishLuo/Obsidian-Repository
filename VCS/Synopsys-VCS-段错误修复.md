---
uid: fixSegfaultVCS
title: Synopsys VCS 段错误修复
tags:
  - VCS
  - Synopsys
  - segfault
  - GDB
  - binary-patch
  - 32bit
  - 调试
created: 2026-06-05
source: https://github.com/mperov/fixSegfaultVCS
description: 通过 GDB 调试和二进制补丁修复 Synopsys VCS 仿真器的 segfault 问题
---

## 背景

VCS Q-2020.03（64位）和 R-2020.12-SP1（32位）在运行时会产生 segfault，问题根源相同：某个 SNPSle 函数调用后没有返回有效数据，导致 `mem_free` 崩溃。

## 通用方法论

### 1. GDB 调试定位问题（64位）

```bash
gdb ./simv
run
# 程序崩溃后
backtrace
#0  0x00002aaacb911904 in mem_free () from .../libsnpsmalloc.so
#6  0x00002aaacd8418e3 in SNPSle_126d17af453e054a () from .../libvcsnew.so
frame 6
disas
```

注意 frame 6 中的 `callq 0x2aaacd8416d6 <SNPSle_2c2186f0f2f42415>`，这个被调函数没有 return data。

### 2. objdump 反汇编定位字节

```bash
objdump -D /path/to/vcs/linux64/lib/libvcsnew.so &> dump
# 用 vim 打开，搜索出问题的函数名
```

64位关键行（`1aaf8de`）：

```asm
1aaf8de:   e8 f3 fd ff ff          callq  1aaf6d6 <SNPSle_2c2186f0f2f42415>
```

32位关键行（`19e2771`）：

```asm
19e2771:   e8 b0 fd ff ff          call   19e2526 <SNPSle_59279f7c9d103b1e>
```

### 3. hexedit 打补丁

> ⚠️ **打补丁前必须先备份 VCS！**

```bash
hexedit /path/to/vcs/linux64/lib/libvcsnew.so
```

64位：找到 `e8 f3 fd ff ff`（地址 `1aaf8de`），替换为 `0x90`
32位：找到 `e8 b0 fd ff ff`（地址 `19e2771`），替换为 `0x90`

### 4. 验证

```bash
objdump -D /path/to/vcs/linux64/lib/libvcsnew.so &> new_dump
# 确认原来 callq 指令的位置已被 NOP(0x90) 替代
```

---

## 案例一：64位 VCS Q-2020.03

### 问题

```bash
Command line: ./simv

--- Stack trace follows:

Dumping VCS Annotated Stack:
#0  0x00002aaad3587659 in waitpid () from /lib64/libc.so.6
#1  0x00002aaad3504f62 in do_system () from /lib64/libc.so.6
...
#9  0x00002aaacb911904 in mem_free () from .../libsnpsmalloc.so
#10 0x00002aaacb8ed015 in snpsCheckFreeFunc () from .../libsnpsmalloc.so
#11 0x00002aaacd8428f4 in SNPSle_540905a7b1b27611 () from .../libvcsnew.so
#12 0x00002aaacd84323a in SNPSle_9ff9b2dce8f1fde9 () from .../libvcsnew.so
#13 0x00002aaacd841308 in SNPSle_36c2d46996c96605 () from .../libvcsnew.so
#14 0x00002aaacd8417cd in SNPSle_2c2186f0f2f42415 () from .../libvcsnew.so
#15 0x00002aaacd8418e3 in SNPSle_126d17af453e054a () from .../libvcsnew.so
#16 0x00002aaad2ca3ea5 in start_thread () from /lib64/libpthread.so.0
#17 0x00002aaad35c0b0d in clone () from /lib64/libc.so.6
#0  0x00002aaad35879fd in nanosleep () from /lib64/libc.so.6
...
#18 0x00000000004171ff in main ()

Process VmPeak: 916968 kb, VmSize: 851436 kb
```

### GDB 调试过程

```bash
gdb ./simv
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-120.el7
...
Reading symbols from /test/run/simv...(no debugging symbols found)...done.
(gdb) run

Program received signal SIGSEGV, Segmentation fault.
0x00002aaacb911904 in mem_free () from .../libsnpsmalloc.so
(gdb) backtrace
#0  ... in mem_free ()
#1  ... in snpsCheckFreeFunc ()
#2  ... in SNPSle_540905a7b1b27611 ()
#3  ... in SNPSle_9ff9b2dce8f1fde9 ()
#4  ... in SNPSle_36c2d46996c96605 ()
#5  ... in SNPSle_2c2186f0f2f42415 ()
#6  ... in SNPSle_126d17af453e054a ()
(gdb) frame 6
(gdb) disas
Dump of assembler code for function SNPSle_126d17af453e054a:
   0x00002aaacd8418c0 <+0>:    push   %rbx
   0x00002aaacd8418c1 <+1>:    mov    %rdi,%rbx
   0x00002aaacd8418c4 <+4>:    xor    %edi,%edi
   0x00002aaacd8418c6 <+6>:    test   %rbx,%rbx
   0x00002aaacd8418c9 <+9>:    je     0x2aaacd8418de <+30>
   0x00002aaacd8418cb <+11>:  mov    (%rbx),%eax
   0x00002aaacd8418cd <+13>:  test   %eax,%eax
   0x00002aaacd8418cf <+15>:  je     0x2aaacd841930 <+112>
   0x00002aaacd8418d1 <+17>:  imul   $0xf4240,%eax,%edi
   0x00002aaacd8418d7 <+23>:  callq  0x2aaacce069c0 <usleep@plt>
   0x00002aaacd8418dc <+28>:  jmp    0x2aaacd841930 <+112>
   0x00002aaacd8418de <+30>:  callq  0x2aaacd8416d6 <SNPSle_2c2186f0f2f42415>
=> 0x00002aaacd8418e3 <+35>:  dec    %eax
   ...
```

`SNPSle_2c2186f0f2f42415` 没有 return data，可以忽略。

### objdump 找到关键字节

```bash
objdump -D /eda/synopsys/vcs/Q-2020.03/linux64/lib/libvcsnew.so &> dump
# vim 搜索 SNPSle_126d17af453e054a

0000000001aaf8c0 <SNPSle_126d17af453e054a>:
 1aaf8c0:   53                      push   %rbx
 1aaf8c1:   48 89 fb                mov    %rdi,%rbx
 1aaf8c4:   31 ff                   xor    %edi,%edi
 1aaf8c6:   48 85 db                test   %rbx,%rbx
 1aaf8c9:   74 13                   je     1aaf8de<+0x1e>
 1aaf8cb:   8b 03                   mov    (%rbx),%eax
 1aaf8cd:   85 c0                   test   %eax,%eax
 1aaf8cf:   74 5f                   je     1aaf930 <+0x70>
 1aaf8d1:   69 f8 40 42 0f 00       imul   $0xf4240,%eax,%edi
 1aaf8d7:   e8 e4 50 5c ff          callq  10749c0 <usleep@plt>
 1aaf8dc:   eb 52                   jmp    1aaf930 <+0x70>
 1aaf8de:   e8 f3 fd ff ff          callq  1aaf6d6 <SNPSle_2c2186f0f2f42415>
 1aaf8e3:   ff c8                   dec    %eax
 1aaf8e5:   48 8d 05 14 56 b5 03    lea    ...(%rip),%rax
```

关键行：`1aaf8de: e8 f3 fd ff ff` — 将此 callq 指令替换为 5 个 NOP（`0x90`）。

### 验证

```bash
objdump -D /eda/synopsys/vcs/Q-2020.03/linux64/lib/libvcsnew.so &> new_dump
# 查看 1aaf8de 处应为 90 90 90 90 90
```

```bash
./simv
$finish called from file ".../uvm_root.svh", line 613.
$finish at simulation time 0.000ns
           V C S   S i m u l a t i o n   R e p o r t
Time: 0 ps
CPU Time: 2.420 seconds
```

---

## 案例二：32位 VCS R-2020.12-SP1（无 -full64）

### 问题

```bash
Command line: ./simv

Dumping VCS Annotated Stack:
#0  0xf7ffd430 in __kernel_vsyscall ()
#1  0x006d569b in waitpid () from /lib/libc.so.6
#2  0x0066ebe3 in do_system () from /lib/libc.so.6
#3  0x0066ef72 in system () from /lib/libpthread.so.0
#4  0x007db9ed in system () from /lib/libpthread.so.0
#8  0xea2beb40 in SNPSle_64133461705005bb725549e2e6fa1b3f () from .../libvcsnew.so
#9  0xe9f6c74c in SNPSle_82244d58c54c18c70d63edc9becab634 () from .../libvcsnew.so
#11 0xee448195 in mem_free () from .../libsnpsmalloc.so
#12 0xee420bc4 in snpsFreeFunc () from .../libsnpsmalloc.so
#13 0xe9ce58ee in SNPSle_edfd0437dc1f3ca1 () from .../libvcsnew.so
...
#17 0xe9ce4776 in SNPSle_207fb97ffa75eca3 () from .../libvcsnew.so
#18 0x007d3bc9 in start_thread () from /lib/libpthread.so.0
#19 0x0071711e in clone () from /lib/libc.so.6

Process VmPeak: 553656 kb, VmSize: 421564 kb
```

### objdump 找到关键字节

```bash
objdump -D /eda/SYNOPSYS/vcs/R-2020.12-SP1/linux/lib/libvcsnew.so &> dump
# vim 搜索 SNPSle_207fb97ffa75eca3

019e273a <SNPSle_207fb97ffa75eca3>:
 19e273a:   55                      push   %ebp
 19e273b:   31 c0                   xor    %eax,%eax
 19e273d:   89 e5                   mov    %esp,%ebp
 19e273f:   56                      push   %esi
 19e2740:   8b 75 08                mov    0x8(%ebp),%esi
 19e2743:   53                      push   %ebx
 19e2744:   e8 c6 14 5c 02          call   3fa3c0f <...>
 19e2749:   81 c3 b7 58 28 04       add    $0x42858b7,%ebx
 19e274f:   85 f6                   test   %esi,%esi
 19e2751:   74 1a                   je     19e276d <+0x33>
 19e2753:   8b 06                   mov    (%esi),%eax
 19e2755:   85 c0                   test   %eax,%eax
 19e2757:   74 63                   je     19e27bc <+0x82>
 19e2759:   69 c0 40 42 0f 00       imul   $0xf4240,%eax,%eax
 19e275f:   83 ec 0c                sub    $0xc,%esp
 19e2762:   50                      push   %eax
 19e2763:   e8 28 03 55 ff          call   f32a90 <usleep@plt>
 19e2768:   83 c4 10                add    $0x10,%esp
 19e276b:   eb 4f                   jmp    19e27bc <+0x82>
 19e276d:   83 ec 0c                sub    $0xc,%esp
 19e2770:   50                      push   %eax
 19e2771:   e8 b0 fd ff ff          call   19e2526 <SNPSle_59279f7c9d103b1e>
 19e2776:   83 c4 10                add    $0x10,%esp
 19e2779:   48                      dec    %eax
  ...
```

关键行：`19e2771: e8 b0 fd ff ff` — 将此 callq 指令替换为 5 个 NOP（`0x90`）。

### 验证后

```asm
19e2771:   90                      nop
19e2772:   90                      nop
19e2773:   90                      nop
19e2774:   90                      nop
19e2775:   90                      nop
19e2776:   83 c4 10                add    $0x10,%esp
```

---

## 自动化补丁 patcher.py

自动搜索并替换 libvcsnew.so 中的问题字节，无需手动 hexedit。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import sys

BUF_SIZE = 1024
ORIGIN_BYTES = b'\xe8\xf3\xfd\xff\xff\xff\xc8'
NEW_BYTES = b'\x90\x90\x90\x90\x90'

def patch(path2lib, addresses):
    with open(path2lib, 'r+b') as fh:
        for address in addresses:
            fh.seek(address)
            fh.write(NEW_BYTES)
            print(path2lib + ' is patched by ' + str(NEW_BYTES) + ' on position - ' + hex(address))

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("usage: ./patcher.py /path/to/library.so")
        sys.exit(-1)

    path2lib = sys.argv[1]

    try:
        with open(path2lib, 'rb') as fh:
            first4 = fh.read(4)
            if first4 != b'\x7fELF':
                print(path2lib + " isn't ELF file!")
                sys.exit(-1)
    except Exception as e:
        print(str(e))
        sys.exit(-1)

    print("library - " + path2lib)

    addresses = []
    address = 0
    count_match = 0
    with open(path2lib, 'rb') as fh:
        while True:
            buf = fh.read(BUF_SIZE)
            if buf:
                for byte in buf:
                    if byte == ORIGIN_BYTES[count_match]:
                        count_match += 1
                        if count_match == len(ORIGIN_BYTES):
                            print(str(ORIGIN_BYTES) + ' is found!')
                            addr = address - len(ORIGIN_BYTES) + 1
                            print('Address is ' + hex(addr))
                            addresses.append(addr)
                            count_match = 0
                    else:
                        count_match = 0
                    address += 1
            else:
                break

    patch(path2lib, addresses)
```

> ⚠️ 运行 patcher.py 前**必须先备份**要打补丁的库文件！

---

## 相关文档

[[GDB/GDB-urg-调试.md]] — GDB 调试 urg1 案例，含 --args 参数和 LD_LIBRARY_PATH 设置方法

## 关键字节对照表

| 版本 | 库文件 | 目标地址 | 待替换字节 | 替换为 |
| --- | --- | --- | --- | --- |
| Q-2020.03 (64bit) | libvcsnew.so | 1aaf8de | `e8 f3 fd ff ff` | `90 90 90 90 90` |
| R-2020.12-SP1 (32bit) | libvcsnew.so | 19e2771 | `e8 b0 fd ff ff` | `90 90 90 90 90` |

