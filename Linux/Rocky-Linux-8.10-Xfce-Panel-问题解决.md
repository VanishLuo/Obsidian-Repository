---
tags: [rocky-linux, xfce, panel]
---

# Rocky Linux 8.10 Xfce Panel 问题解决

## 问题一：同一应用程序窗口叠加显示

### 问题描述

在 Rocky Linux 8.10 的 Xfce 桌面环境中，Panel（任务栏）会自动将同一应用程序的多个窗口叠加分组显示。这会导致用户在操作多个同类窗口时不够直观，例如同时打开多个文件管理器窗口时，它们会堆叠在一起，无法直接看到所有窗口的标题。

### 解决方法

1. 在 Panel 上**右键点击**，选择 `Panel Preferences...`
2. 切换到 `Items` 选项卡
3. 找到并选择 `Window Buttons` 插件，点击右侧的设置图标
4. 在 `Behaviour` 设置区域找到 `window grouping` 选项
5. 将其设置为 **`Never`**

### 效果

设置完成后，同一应用程序的多个窗口将不再自动分组，每个窗口会独立显示在 Panel 上，可直接点击切换，方便多窗口操作与管理。

---

## 问题二：Panel 控件无法点击

### 问题现象

Panel 上的所有控件（如终端、菜单图标、插件按钮等）突然无法点击，无法使用。

### 解决方案

```bash
killall -9 xfce4-panel
xfce4-panel &
```

### 说明

通过强制终止并重新启动 xfce4-panel 进程来恢复 Panel 的正常功能。
