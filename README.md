# WinDbg CDB 调试技能

一个用于通过 PowerShell 使用 CDB（控制台调试器）进行 Windows 程序调试的 AI 技能。

## 📋 目录

- [简介](#简介)
- [前置条件](#前置条件)
- [⚠️ 重要：路径配置](#-重要路径配置)
- [安装](#安装)
- [使用方法](#使用方法)
- [注意事项](#注意事项)
- [常见问题](#常见问题)

---

## 简介

本技能提供了完整的 CDB（Console Debugger）调试方案，通过 PowerShell 进行交互，无需依赖 MCP 工具。适用于：

- 调试 Windows 可执行文件
- 分析崩溃转储文件（.dmp）
- 设置断点、反汇编、检查内存/寄存器/调用栈
- 附加到正在运行的进程进行调试
- 远程调试会话
- 任何 Windows 动态调试需求

---

## 前置条件

### 必需软件

在使用本技能之前，您需要准备以下内容：

#### 1. WinDbg Preview（包含 CDB）

**安装方法：**

```powershell
# 使用 winget 安装（推荐）
winget install "WinDbg"

# 或从 Microsoft Store 下载 WinDbg Preview
```

**验证安装：**

安装完成后，CDB 可执行文件应位于以下路径之一：
- `C:\Users\[你的用户名]\AppData\Local\Microsoft\WindowsApps\cdbX64.exe`（64位）
- `C:\Users\[你的用户名]\AppData\Local\Microsoft\WindowsApps\cdbX86.exe`（32位）
- `C:\Users\[你的用户名]\AppData\Local\Microsoft\WindowsApps\cdbARM64.exe`（ARM64）

#### 2. PowerShell

Windows 10/11 系统自带，确保使用 PowerShell 5.1 或更高版本。

#### 3.（可选）调试符号

- 本技能**不使用**远程符号服务器
- 仅支持本地符号文件（.pdb）
- 对于自己的程序，编译时使用 `-g`（GCC/MinGW）或 `/DEBUG`（MSVC）生成符号

### 系统要求

- Windows 10 或更高版本
- 管理员权限（某些调试操作需要）
- 至少 2GB 可用内存

---

## ⚠️ 重要：路径配置

**在使用本技能之前，您必须修改技能文件中的绝对路径！**

技能文件中的 CDB 路径默认设置为：

```powershell
$cdb64 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
$cdb86 = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbX86.exe"
$cdbArm = "C:\Users\j4543\AppData\Local\Microsoft\WindowsApps\cdbARM64.exe"
```

### 修改步骤

1. **查找您的 CDB 路径**

```powershell
# 在 PowerShell 中运行以下命令查找 CDB 位置
Get-ChildItem -Path "C:\Users\$env:USERNAME\AppData\Local\Microsoft\WindowsApps" -Filter "cdb*.exe"
```

2. **更新路径**

将找到的实际路径替换到以下文件中的路径：
- `windbg-cdb-powershell/SKILL.md`

### 替换示例

如果您的用户名是 `YourName`，则应将路径修改为：

```powershell
$cdb64 = "C:\Users\YourName\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
$cdb86 = "C:\Users\YourName\AppData\Local\Microsoft\WindowsApps\cdbX86.exe"
$cdbArm = "C:\Users\YourName\AppData\Local\Microsoft\WindowsApps\cdbARM64.exe"
```

---

## 安装

1. 克隆或下载本项目到本地

```powershell
git clone https://github.com/CuteCuteYu/cdb_skill_debug_windbg.git
cd cdb_skill_debug_windbg
```

2. 确保 CDB 已正确安装并更新了路径配置（参见上方 [路径配置](#-重要路径配置)）

3. 将技能文件放置到您的 AI 助手技能目录

---

## 使用方法

### 基本调试流程

#### 1. 调试可执行文件

```powershell
$cdb64 = "C:\Users\[您的用户名]\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

# 启动程序进行调试
& $cdb64 "program.exe"

# 使用命令启动后退出
& $cdb64 -c "x program!*; q" "program.exe"
```

#### 2. 分析崩溃转储

```powershell
$cdb64 = "C:\Users\[您的用户名]\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

# 分析转储文件
& $cdb64 -z "crash.dmp" -c "!analyze -v; q"
```

#### 3. 附加到运行中的进程

```powershell
$cdb64 = "C:\Users\[您的用户名]\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"

# 通过 PID 附加
& $cdb64 -p 1234

# 通过进程名附加
$pid = (Get-Process notepad).Id
& $cdb64 -p $pid
```

### 常用 CDB 命令

| 命令 | 功能 |
|------|------|
| `g` | 继续执行 |
| `p` | 单步执行（跳过函数调用） |
| `t` | 单步执行（进入函数调用） |
| `bp 地址` | 设置断点 |
| `bl` | 列出所有断点 |
| `bc *` | 清除所有断点 |
| `k` | 显示调用栈 |
| `r` | 显示寄存器 |
| `dv` | 显示局部变量 |
| `uf 函数名` | 反汇编函数 |

---

## 注意事项

### 1. 编码规则（非常重要）

- **CDB 脚本文件必须使用 ASCII 编码**
- **CDB 命令和注释只能使用英文**
- **不要在 CDB 命令中使用中文字符**

正确的写法：
```powershell
$script = @'
$$ Set breakpoint on MessageBoxA
bp user32!MessageBoxA ".echo *** CALLED ***; g"
g
'@
$script | Out-File -FilePath "debug.txt" -Encoding ASCII
```

错误的写法：
```powershell
$$ 在MessageBoxA下断点
bp user32!MessageBoxA ".echo 调用; g"
```

### 2. 架构匹配

- 使用 `cdbX64.exe` 调试 64 位程序
- 使用 `cdbX86.exe` 调试 32 位程序
- 使用 `cdbARM64.exe` 调试 ARM64 程序

### 3. 符号服务器限制

本技能**不使用** Microsoft 的公共符号服务器，原因如下：
- 需要网络访问
- 符号下载可能很慢且很大
- 离线调试更可靠快速

在没有远程符号的情况下，仍然可以：
- 使用导出函数名（如 `user32!MessageBoxA`）
- 使用自己模块的符号
- 使用原始地址和偏移量
- 完整的寄存器状态和调用栈

### 4. 输出处理

使用 `2>$null` 抑制 CDB 的 NatVis 卸载消息：

```powershell
& $cdb64 -c "command; q" "program.exe" 2>$null
```

---

## 常见问题

### Q: 提示找不到 CDB 可执行文件？

**A:** 请确保：
1. 已安装 WinDbg Preview
2. 已正确修改技能文件中的路径配置
3. 使用与您的用户名匹配的实际路径

### Q: CDB 启动后提示"bad image"错误？

**A:** 这通常是架构不匹配导致的：
- 如果是 32 位程序，使用 `cdbX86.exe`
- 如果是 64 位程序，使用 `cdbX64.exe`

### Q: 为什么看不到系统 DLL 的详细符号信息？

**A:** 本技能不使用远程符号服务器，因此只能看到：
- 导出函数名（如 `user32!MessageBoxA`）
- 原始地址
- 模块名称和加载地址

对于大多数调试任务，这些信息已经足够。

### Q: 如何确定程序是 32 位还是 64 位？

**A:** 使用以下 PowerShell 命令：

```powershell
$cdb64 = "C:\Users\[您的用户名]\AppData\Local\Microsoft\WindowsApps\cdbX64.exe"
& $cdb64 -c "lm; q" "program.exe" 2>$null
```

如果出现"bad image"错误，则程序可能是 32 位的。

---

## 许可证

本项目采用 MIT 许可证。详见 [LICENSE](LICENSE) 文件。

---

## 贡献

欢迎提交 Issue 和 Pull Request！

---

## 相关资源

- [WinDbg Preview 文档](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools)
- [CDB 命令参考](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-commands)
