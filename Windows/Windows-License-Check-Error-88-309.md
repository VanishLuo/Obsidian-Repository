---
tags: [Windows, License, Error, 日期时间, FLEXnet]
---

# Windows License Check Error -88,309 解决方法

## 核心问题

安装或启动软件时遇到 `License Check Error -88,309`，表示**系统日期被设置在未来时间**，许可证服务器校验失败。这是 FLEXnet 许可证管理系统的组合错误码格式。

## 错误信息

```plaintext
License Check Error -88,309
License server clock is incorrect
or
License server date has not yet been reached
```

## 原因分析

| 原因 | 说明 |
| --- | --- |
| **系统日期被设置为未来** | 用户可能调整过系统时间/日期，或 BIOS 时钟出错 |
| **虚拟机时间不同步** | 虚拟机与宿主机时间不同步 |
| **BIOS 硬件时钟错误** | 主板纽扣电池电量不足导致时钟走慢/走快 |
| **FLEXnet 时间校验** | 许多商业软件（MATLAB、Altair、Siemens、NVIDIA 等）使用 FLEXnet，对时间敏感 |

## 解决方法

### 方法一：检查并修正系统日期和时间（最常见）

1. **自动修复**：
   - 右键右下角时间 → 「调整日期和时间」
   - 开启「自动设置时间」开关，Windows 会自动同步 Internet 时间

2. **手动修正**：
   - 关闭「自动设置时间」
   - 点击「更改」手动设置正确的日期和时间（确认是当前实际时间）
   - 确认时区和日期正确

3. **强制同步 Internet 时间**：

```powershell
# 以管理员身份运行 PowerShell
net start W32time
w32tm /resync /force
```

### 方法二：检查 BIOS 时钟

如果 Windows 日期正确但问题仍然存在，可能是 BIOS 硬件时钟出错：

1. 重启电脑
2. 进入 BIOS 设置（通常是 `Del` 或 `F2`）
3. 检查硬件时钟（RTC Clock）是否正确
4. 如果 BIOS 时间也不对：
   - 更换主板纽扣电池（CR2032）
   - 在 BIOS 中手动设置正确时间

### 方法三：虚拟机时间同步

如果是虚拟机环境：

| 虚拟机软件 | 解决方法 |
| --- | --- |
| **VMware** | 虚拟机设置 → 选项 → 高级 → 取消「与主机同步时间」 |
| **Hyper-V** | 虚拟机设置 → 集成服务 → 取消「时间同步」 |
| **VirtualBox** | 虚拟机设置 → 系统 → 取消「启用时间同步」 |

### 方法四：处理 FLEXnet 许可证服务

```powershell
# 检查 FLEXnet 相关服务状态
Get-Service | Where-Object { $_.DisplayName -like "*FLEX*" -or $_.Name -like "*lm*" }

# 重启许可证服务（需要管理员权限）
Restart-Service "服务名称"

# 查看许可证服务器日志
# 通常位于 C:\Program Files\xxx\license.log 或 C:\Program Files\xxx\Logs\
```

## 快速排查流程

```plaintext
1. 检查系统日期和时间是否正确（是否为当前时间，非未来）
       ↓
2. 如果日期正确 → 检查 BIOS 硬件时钟
       ↓
3. 如果 BIOS 也正确 → 检查是否是虚拟机时间同步问题
       ↓
4. 修正时间后 → 重启软件或重新加载许可证服务
```

## 相关 FLEXnet 错误码速查

| 错误码 | 含义 |
| --- | --- |
| `-88,309` | 系统日期在未来，许可证服务器校验失败 |
| `-97` | 许可证文件找不到或损坏 |
| `-96` | 许可证服务器不可用 |
| `-10` | 许可证文件格式错误 |
| `-100` | 未找到有效的许可证 |

## 预防措施

1. **启用 Windows 自动时间同步**：保持系统时间准确
2. **避免手动调整系统时间**：调后再调回可能导致许可证缓存出错
3. **虚拟机用户**：虚拟机启动后手动同步一次时间
4. **定期检查**：定期检查系统时间是否准确

## 提示

- `-88,309` 中 `-88` 表示时间校验失败，`309` 是 FLEXnet 的扩展错误码，两者组合说明是许可证服务器检测到系统日期在未来
- 修正时间后建议重启软件，以确保重新加载许可证验证
