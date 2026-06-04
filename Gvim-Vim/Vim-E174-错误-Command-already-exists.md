---
tags: [gvim, vim]
---

# Vim/gVim E174 错误：Command already exists

## 错误信息

```text
Error detected while processing /path/to/script.vim:
line  11:
E174: Command already exists: add ! to replace it
```

当 Vim 尝试重复定义同名自定义命令时触发。Vim 默认阻止重复定义以避免意外覆盖。

## 定位命令定义

使用以下命令查看命令定义的位置：

```vim
:verbose command MANPAGER
```

输出示例会显示命令在哪个文件第几行被定义，包括是否使用了 `command!`。

## 实际场景

用户检查后发现 `~/.vim/plugin/manpager.vim` 中已经使用了 `command!`，但仍报 E174，说明问题可能来自：

- 系统的 `/usr/local/share/vim/vim91/plugin/manpager.vim`
- 其他插件或配置文件中的同名命令

## 解决方案

### 方案一：删除用户目录下的重复插件（最直接）

系统插件已完整，删除用户那份即可：

```bash
rm ~/.vim/plugin/manpager.vim
```

### 方案二：禁用问题插件

在 `~/.vimrc` 文件开头添加 `g:loaded_xxx_plugin = 1`：

```vim
" ~/.vimrc 文件开头
let g:loaded_manpager_plugin = 1
```

### 方案三：继续排查真正的冲突源

用户的 `manpager.vim` 已使用 `command!`，说明冲突来自其他地方：

1. 用 `:verbose command MANPAGER` 找到报错实际指向的文件和行数
2. 如果指向的是系统插件或其他插件，检查其使用的是 `command` 还是 `command!`
3. 如有需要，向系统插件或对应插件作者提交 issue 或等待修复

### 方案四：等待系统或插件更新

如果是系统插件的问题，可能需要等系统更新 Vim 版本或插件修复后重新加载。

---

## 常见触发场景

1. **重复 source**：手动执行 `:source $MYVIMRC` 导致配置被加载两次
2. **插件冲突**：多插件定义了同名命令
3. **配置与插件冲突**：vimrc 中的命令与插件命令同名
4. **插件 bug**：老旧插件未使用 `command!`

---

## 参考

- [amix/vimrc#239](https://github.com/amix/vimrc/issues/239)
- [VundleVim/Vundle.vim#784](https://github.com/VundleVim/Vundle.vim/issues/784)
- [llvm/llvm-project#82410](https://github.com/llvm/llvm-project/pull/82410)
