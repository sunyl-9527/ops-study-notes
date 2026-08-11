# PowerShell 学习笔记

> 适用对象：希望系统学习 Windows PowerShell / PowerShell 7，并用于日常 Windows 运维、自动化脚本和云计算运维辅助工作的学习者。
>
> 学习目标：理解 PowerShell 的对象管道思想，掌握常用命令、脚本编写、模块管理、远程管理和常见运维自动化场景。

---

## 一、PowerShell 是什么

PowerShell 是微软推出的命令行 Shell 和脚本语言，主要用于系统管理、自动化运维、批量处理任务和云平台管理。

它和传统 `cmd` 最大的区别是：

- `cmd` 更偏字符串命令行工具。
- `PowerShell` 更偏对象化自动化平台。
- PowerShell 管道中传递的通常不是纯文本，而是结构化对象。

一句话理解：PowerShell 不只是执行命令，而是把系统里的进程、服务、文件、事件日志、注册表、网络配置等资源当成对象来查询、筛选和操作。

---

## 二、版本与环境

### 1. Windows PowerShell 与 PowerShell 7

| 名称 | 说明 | 常见命令 |
|---|---|---|
| Windows PowerShell | Windows 自带版本，通常是 5.1 | `powershell` |
| PowerShell 7+ | 跨平台新版，支持 Windows / Linux / macOS | `pwsh` |

建议：

- Windows 自带的 `Windows PowerShell 5.1` 可以满足大部分系统管理需求。
- 如果要学习长期使用，建议同时安装 `PowerShell 7`。
- 如果要做跨平台脚本，优先使用 `PowerShell 7`。

### 2. 查看版本

```powershell
$PSVersionTable
```

重点关注：

- `PSVersion`：PowerShell 版本。
- `PSEdition`：`Desktop` 通常代表 Windows PowerShell，`Core` 通常代表 PowerShell 7+。
- `OS`：当前操作系统信息。

### 3. 常用运行入口

```powershell
# 启动 Windows PowerShell
powershell

# 启动 PowerShell 7
pwsh

# 查看帮助
Get-Help
```

---

## 三、命令基础

### 1. Cmdlet 命名规则

PowerShell 命令通常叫 `Cmdlet`，采用 `动词-名词` 格式。

常见动词：

- `Get`：获取信息。
- `Set`：设置属性。
- `New`：新建资源。
- `Remove`：删除资源。
- `Start`：启动。
- `Stop`：停止。
- `Restart`：重启。
- `Test`：测试状态。

示例：

```powershell
Get-Process
Get-Service
Set-Location
New-Item
Remove-Item
Start-Service
Stop-Service
Test-Path
```

### 2. 查看命令

```powershell
# 查找包含 service 的命令
Get-Command *service*

# 查看某个命令的帮助
Get-Help Get-Service

# 查看详细帮助
Get-Help Get-Service -Detailed

# 查看示例
Get-Help Get-Service -Examples

# 在线帮助
Get-Help Get-Service -Online
```

### 3. 常用别名

PowerShell 提供了一些别名，方便从 `cmd` 或 Linux Shell 迁移。

| 别名 | 原命令 | 说明 |
|---|---|---|
| `ls` | `Get-ChildItem` | 查看目录内容 |
| `dir` | `Get-ChildItem` | 查看目录内容 |
| `cd` | `Set-Location` | 切换目录 |
| `pwd` | `Get-Location` | 查看当前位置 |
| `cat` | `Get-Content` | 查看文件内容 |
| `type` | `Get-Content` | 查看文件内容 |
| `cp` | `Copy-Item` | 复制 |
| `mv` | `Move-Item` | 移动或重命名 |
| `rm` | `Remove-Item` | 删除 |
| `echo` | `Write-Output` | 输出内容 |

建议学习时优先记原命令，例如 `Get-ChildItem`，因为原命令语义更清晰，也更适合写脚本。

---

## 四、文件与目录操作

### 1. 路径与目录

```powershell
# 查看当前路径
Get-Location

# 切换目录
Set-Location C:\Projects

# 返回上一级
Set-Location ..

# 查看目录内容
Get-ChildItem

# 查看隐藏文件
Get-ChildItem -Force

# 递归查看
Get-ChildItem -Recurse
```

### 2. 文件创建、复制、移动、删除

```powershell
# 创建目录
New-Item -ItemType Directory -Path .\logs

# 创建文件
New-Item -ItemType File -Path .\app.log

# 复制文件
Copy-Item .\app.log .\logs\app.log

# 移动文件
Move-Item .\app.log .\logs\app.log

# 重命名文件
Rename-Item .\logs\app.log app-2026.log

# 删除文件
Remove-Item .\logs\app-2026.log

# 删除目录及其内容
Remove-Item .\logs -Recurse
```

删除操作要谨慎，尤其是配合 `-Recurse` 时，先用 `Get-ChildItem` 确认目标路径。

### 3. 文件内容读取

```powershell
# 查看完整文件
Get-Content .\app.log

# 查看前 20 行
Get-Content .\app.log -TotalCount 20

# 持续跟踪日志
Get-Content .\app.log -Wait

# 搜索关键字
Select-String -Path .\app.log -Pattern "error"
```

---

## 五、对象与管道

### 1. 管道基础

PowerShell 使用 `|` 把前一个命令的输出传递给后一个命令。

```powershell
Get-Service | Select-Object Name, Status
```

这里传递的不是普通文本，而是服务对象。后面的 `Select-Object` 可以直接读取对象属性。

### 2. 查看对象成员

```powershell
Get-Service | Get-Member
```

常见成员类型：

- `Property`：属性。
- `Method`：方法。
- `ScriptProperty`：脚本属性。

### 3. 筛选对象

```powershell
# 筛选正在运行的服务
Get-Service | Where-Object Status -eq Running

# 筛选 CPU 使用较高的进程
Get-Process | Where-Object CPU -gt 100

# 筛选名称包含 chrome 的进程
Get-Process | Where-Object ProcessName -like "*chrome*"
```

### 4. 选择属性

```powershell
Get-Process | Select-Object ProcessName, Id, CPU
```

### 5. 排序

```powershell
# 按 CPU 使用量倒序
Get-Process | Sort-Object CPU -Descending
```

### 6. 限制输出数量

```powershell
# 查看 CPU 使用最高的前 10 个进程
Get-Process |
    Sort-Object CPU -Descending |
    Select-Object -First 10 ProcessName, Id, CPU
```

---

## 六、变量与数据类型

### 1. 变量

PowerShell 变量以 `$` 开头。

```powershell
$name = "PowerShell"
$count = 10
$enabled = $true
```

### 2. 字符串

```powershell
$name = "Nginx"

# 双引号会解析变量
"Service name is $name"

# 单引号不会解析变量
'Service name is $name'
```

### 3. 数组

```powershell
$services = @("nginx", "mysql", "redis")
$services[0]
$services.Count
```

### 4. 哈希表

```powershell
$config = @{
    Name = "web01"
    Port = 8080
    Env  = "dev"
}

$config.Name
$config["Port"]
```

### 5. 自定义对象

```powershell
$server = [PSCustomObject]@{
    Name = "web01"
    IP   = "192.168.1.10"
    Role = "Nginx"
}

$server.Name
```

---

## 七、条件判断与循环

### 1. if 判断

```powershell
$status = "Running"

if ($status -eq "Running") {
    "Service is running"
} elseif ($status -eq "Stopped") {
    "Service is stopped"
} else {
    "Unknown status"
}
```

常用比较运算符：

| 运算符 | 说明 |
|---|---|
| `-eq` | 等于 |
| `-ne` | 不等于 |
| `-gt` | 大于 |
| `-ge` | 大于等于 |
| `-lt` | 小于 |
| `-le` | 小于等于 |
| `-like` | 通配符匹配 |
| `-match` | 正则匹配 |
| `-contains` | 包含 |

### 2. foreach 循环

```powershell
$services = @("wuauserv", "bits", "spooler")

foreach ($service in $services) {
    Get-Service -Name $service
}
```

### 3. 管道循环

```powershell
Get-Service | ForEach-Object {
    "$($_.Name) - $($_.Status)"
}
```

`$_` 表示管道中当前正在处理的对象。

---

## 八、脚本基础

### 1. 脚本文件

PowerShell 脚本文件扩展名是 `.ps1`。

示例：`check-service.ps1`

```powershell
param(
    [string]$Name = "spooler"
)

$service = Get-Service -Name $Name -ErrorAction SilentlyContinue

if ($null -eq $service) {
    Write-Host "Service not found: $Name"
    exit 1
}

Write-Host "Service: $($service.Name), Status: $($service.Status)"
```

运行脚本：

```powershell
.\check-service.ps1 -Name spooler
```

### 2. 执行策略

Windows 默认可能限制脚本执行。

```powershell
# 查看当前执行策略
Get-ExecutionPolicy

# 查看所有作用域的执行策略
Get-ExecutionPolicy -List

# 仅对当前用户放宽策略
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

常见策略：

- `Restricted`：禁止执行脚本。
- `RemoteSigned`：本地脚本可执行，远程下载脚本需要签名。
- `Unrestricted`：限制较少，不建议长期使用。
- `Bypass`：绕过策略，多用于临时自动化场景。

### 3. 参数与函数

```powershell
function Get-AppServiceStatus {
    param(
        [Parameter(Mandatory = $true)]
        [string]$Name
    )

    Get-Service -Name $Name | Select-Object Name, Status, StartType
}

Get-AppServiceStatus -Name spooler
```

---

## 九、系统运维常用命令

### 1. 进程管理

```powershell
# 查看进程
Get-Process

# 按名称查找进程
Get-Process -Name chrome

# 停止进程
Stop-Process -Name notepad

# 根据 PID 停止进程
Stop-Process -Id 1234
```

### 2. 服务管理

```powershell
# 查看服务
Get-Service

# 查看指定服务
Get-Service -Name spooler

# 启动服务
Start-Service -Name spooler

# 停止服务
Stop-Service -Name spooler

# 重启服务
Restart-Service -Name spooler
```

### 3. 事件日志

```powershell
# 查看系统日志最近 20 条（旧接口，仅支持经典日志，PowerShell 7 中已不可用）
Get-EventLog -LogName System -Newest 20

# 查看应用日志中的错误
Get-EventLog -LogName Application -EntryType Error -Newest 20

# 推荐使用的新版事件日志命令，性能更好，兼容经典日志和新版 ETW 日志
Get-WinEvent -LogName System -MaxEvents 20

# 按条件筛选，比 Get-EventLog 的过滤能力更强
Get-WinEvent -FilterHashtable @{ LogName = 'System'; Level = 2; StartTime = (Get-Date).AddDays(-1) }
```

`Get-EventLog` 是 Windows PowerShell 5.1 时代的旧接口，只能读取经典事件日志，在 PowerShell 7+ 中已被移除；新脚本应优先使用 `Get-WinEvent`。

### 4. 网络排查

```powershell
# 查看 IP 配置
Get-NetIPConfiguration

# 查看网卡
Get-NetAdapter

# 测试端口连通性
Test-NetConnection example.com -Port 443

# 查看 TCP 连接
Get-NetTCPConnection

# 查看监听端口
Get-NetTCPConnection -State Listen
```

### 5. 磁盘与系统信息

```powershell
# 查看磁盘卷
Get-Volume

# 查看物理磁盘
Get-PhysicalDisk

# 查看计算机信息
Get-ComputerInfo

# 查看系统启动时间
(Get-CimInstance Win32_OperatingSystem).LastBootUpTime
```

### 6. 计划任务

```powershell
# 查看所有计划任务
Get-ScheduledTask

# 查看指定计划任务的运行状态
Get-ScheduledTaskInfo -TaskName "MyBackupTask"

# 创建一个每天凌晨 2 点执行的计划任务
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-File C:\scripts\backup.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 2am
Register-ScheduledTask -TaskName "MyBackupTask" -Action $action -Trigger $trigger -User "SYSTEM"

# 手动触发一次
Start-ScheduledTask -TaskName "MyBackupTask"

# 禁用/启用
Disable-ScheduledTask -TaskName "MyBackupTask"
Enable-ScheduledTask -TaskName "MyBackupTask"

# 删除
Unregister-ScheduledTask -TaskName "MyBackupTask" -Confirm:$false
```

计划任务是 Windows 上等价于 Linux `cron` 的机制，适合把巡检、备份、清理脚本设置为定时自动执行。

---

## 十、文本处理与格式化输出

### 1. 搜索文本

```powershell
Select-String -Path .\app.log -Pattern "error"
```

### 2. 输出表格

```powershell
Get-Service | Format-Table Name, Status, StartType -AutoSize
```

### 3. 输出列表

```powershell
Get-Process -Name explorer | Format-List *
```

### 4. 导出 CSV

```powershell
Get-Service | Select-Object Name, Status, StartType | Export-Csv .\services.csv -NoTypeInformation
```

### 5. 读取 CSV

```powershell
Import-Csv .\services.csv
```

### 6. JSON 处理

```powershell
$data = @{
    name = "web01"
    role = "nginx"
    port = 80
}

$json = $data | ConvertTo-Json
$json | ConvertFrom-Json
```

---

## 十一、模块与包管理

### 1. 查看模块

```powershell
# 查看已加载模块
Get-Module

# 查看可用模块
Get-Module -ListAvailable
```

### 2. 安装模块

```powershell
# 查看 PowerShellGet
Get-Module PowerShellGet -ListAvailable

# 查找模块
Find-Module Az

# 安装模块到当前用户
Install-Module Az -Scope CurrentUser
```

### 3. 导入模块

```powershell
Import-Module Az
```

常见模块：

- `Az`：管理 Azure 云资源。
- `ActiveDirectory`：管理 AD 域资源。
- `Pester`：PowerShell 测试框架。
- `PSReadLine`：命令行编辑增强。

### 4. Windows 系统级包管理（WinGet）

`winget` 是 Windows 自带的系统级包管理器，用来安装日常工具软件，和 PowerShell 模块（`Install-Module`）不是一回事：

```powershell
# 搜索软件包
winget search git

# 安装软件包
winget install --id Git.Git -e

# 查看已安装的软件包
winget list

# 升级所有可升级的软件包
winget upgrade --all
```

### 5. 命令行环境个性化配置

```powershell
# 查看当前用户的 profile 脚本路径（不存在时需要自己创建）
$PROFILE

# 快速编辑 profile
notepad $PROFILE

# 安装/更新 PSReadLine，获得语法高亮、历史搜索、预测建议等增强
Install-Module PSReadLine -Scope CurrentUser -Force

# 在 profile 中开启预测文本（历史命令联想）
Set-PSReadLineOption -PredictionSource History
```

- `$PROFILE` 指向的脚本会在每次打开 PowerShell 时自动执行，适合放常用别名、函数和模块导入。
- 修改 profile 后需要新开一个窗口或 `. $PROFILE` 重新加载才能生效。
- 日常终端建议使用 [Windows Terminal](https://apps.microsoft.com/detail/9n0dx20hk701)（可用 `winget install Microsoft.WindowsTerminal` 安装），比传统控制台支持多标签页、更好的字体渲染和主题配置。

---

## 十二、远程管理

### 1. PowerShell Remoting

PowerShell Remoting 可以远程管理 Windows 主机，底层通常使用 WinRM。

```powershell
# 在目标机器上启用远程管理，需要管理员权限
Enable-PSRemoting -Force

# 测试远程连接
Test-WSMan server01

# 进入远程会话
Enter-PSSession -ComputerName server01

# 执行远程命令
Invoke-Command -ComputerName server01 -ScriptBlock {
    Get-Service
}
```

### 2. 多机器批量执行

```powershell
$servers = @("server01", "server02", "server03")

Invoke-Command -ComputerName $servers -ScriptBlock {
    Get-ComputerInfo | Select-Object CsName, WindowsVersion, OsArchitecture
}
```

远程管理要重点关注：

- 账号权限。
- 防火墙规则。
- WinRM 服务状态。
- 域环境与非域环境认证差异。
- 凭据安全。

---

## 十三、错误处理与调试

### 1. ErrorAction

```powershell
Get-Service -Name not-exist -ErrorAction SilentlyContinue
```

常见取值：

- `Continue`：显示错误并继续。
- `SilentlyContinue`：隐藏错误并继续。
- `Stop`：把错误当成终止错误，便于 `try/catch` 捕获。

### 2. try/catch

```powershell
try {
    Get-Service -Name not-exist -ErrorAction Stop
} catch {
    Write-Host "Failed: $($_.Exception.Message)"
}
```

### 3. 调试输出

```powershell
Write-Host "直接输出到控制台"
Write-Output "输出到管道"
Write-Warning "警告信息"
Write-Error "错误信息"
Write-Verbose "详细信息"
```

使用 `Write-Verbose` 时，需要加 `-Verbose` 才会显示。

```powershell
function Test-Demo {
    [CmdletBinding()]
    param()

    Write-Verbose "Verbose message"
}

Test-Demo -Verbose
```

---

## 十四、安全与权限

### 1. 管理员权限

有些命令需要以管理员身份运行 PowerShell，例如：

- 修改系统服务启动类型。
- 修改防火墙规则。
- 修改系统目录。
- 启用远程管理。
- 安装部分系统级模块。

### 2. 凭据管理

```powershell
# 交互式输入凭据
$cred = Get-Credential

# 在远程命令中使用凭据
Invoke-Command -ComputerName server01 -Credential $cred -ScriptBlock {
    hostname
}
```

不要把明文密码直接写进脚本。需要持久化凭据时，应使用更安全的凭据管理方式，例如 Windows Credential Manager、SecretManagement 模块或平台提供的密钥管理服务。

### 3. 防火墙规则

```powershell
# 查看所有防火墙规则
Get-NetFirewallRule

# 查看已启用的入站规则
Get-NetFirewallRule -Direction Inbound -Enabled True

# 新建一条允许入站的规则（例如放行某端口）
New-NetFirewallRule -DisplayName "Allow-8080" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow

# 禁用/删除规则
Disable-NetFirewallRule -DisplayName "Allow-8080"
Remove-NetFirewallRule -DisplayName "Allow-8080"
```

### 4. Windows Defender

```powershell
# 查看 Defender 当前状态（实时保护、病毒库版本等）
Get-MpComputerStatus

# 触发一次快速扫描
Start-MpScan -ScanType QuickScan

# 查看/更新病毒定义
Get-MpThreatCatalog
Update-MpSignature

# 临时排除某个路径（谨慎使用，扫描时会跳过该路径）
Add-MpPreference -ExclusionPath "C:\dev\workspace"
```

### 5. 脚本安全建议

- 不随意执行来源不明的 `.ps1` 脚本。
- 执行远程下载脚本前先阅读内容。
- 使用最小权限账号运行自动化任务。
- 涉及删除、覆盖、批量修改前先输出待操作对象。
- 生产环境脚本建议记录日志。

---

## 十五、云计算运维中的 PowerShell 场景

### 1. Windows 服务器巡检

```powershell
[PSCustomObject]@{
    Hostname = $env:COMPUTERNAME
    User     = $env:USERNAME
    IP       = (Get-NetIPConfiguration | Where-Object IPv4Address | Select-Object -First 1).IPv4Address.IPAddress
    BootTime = (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
}
```

### 2. 批量查看服务状态

```powershell
$targets = @("spooler", "wuauserv", "bits")

$targets | ForEach-Object {
    Get-Service -Name $_ | Select-Object Name, Status, StartType
}
```

### 3. 端口连通性检查

```powershell
$checks = @(
    @{ Host = "example.com"; Port = 80 },
    @{ Host = "example.com"; Port = 443 }
)

$checks | ForEach-Object {
    Test-NetConnection $_.Host -Port $_.Port | Select-Object ComputerName, RemotePort, TcpTestSucceeded
}
```

### 4. 批量清理临时文件

```powershell
$path = "$env:TEMP"

Get-ChildItem -Path $path -Recurse -Force -ErrorAction SilentlyContinue |
    Where-Object LastWriteTime -lt (Get-Date).AddDays(-7) |
    Remove-Item -Force -Recurse -ErrorAction SilentlyContinue
```

执行批量删除前，可以先去掉最后一段 `Remove-Item`，确认筛选结果是否符合预期。

### 5. Azure 管理入口

```powershell
# 安装 Azure 模块
Install-Module Az -Scope CurrentUser

# 登录 Azure
Connect-AzAccount

# 查看订阅
Get-AzSubscription

# 查看资源组
Get-AzResourceGroup
```

---

## 十六、推荐学习路线

### 第一阶段：命令行基础

目标：能在 PowerShell 中完成日常文件、目录、进程、服务查看。

需要掌握：

- `Get-Command`
- `Get-Help`
- `Get-ChildItem`
- `Set-Location`
- `Get-Content`
- `Select-String`
- `Get-Process`
- `Get-Service`

### 第二阶段：对象与管道

目标：理解 PowerShell 的核心优势，不再只把它当成 `cmd` 使用。

需要掌握：

- `Get-Member`
- `Where-Object`
- `Select-Object`
- `Sort-Object`
- `ForEach-Object`
- `Format-Table`
- `Export-Csv`

### 第三阶段：脚本编写

目标：能写出可复用的小脚本。

需要掌握：

- 变量
- 数组
- 哈希表
- `if`
- `foreach`
- 函数
- `param`
- `try/catch`

### 第四阶段：系统运维

目标：能处理 Windows 主机巡检、服务管理、日志查看和网络排查。

需要掌握：

- `Get-EventLog`
- `Get-WinEvent`
- `Get-NetIPConfiguration`
- `Get-NetAdapter`
- `Test-NetConnection`
- `Get-NetTCPConnection`
- `Get-CimInstance`

### 第五阶段：自动化与远程管理

目标：能批量管理多台 Windows 主机。

需要掌握：

- `Enable-PSRemoting`
- `Enter-PSSession`
- `Invoke-Command`
- `Get-Credential`
- 模块安装与导入
- 日志记录与错误处理

### 第六阶段：云平台与 DevOps 集成

目标：把 PowerShell 用到云资源管理和自动化流程中。

需要掌握：

- Azure `Az` 模块
- 脚本参数化
- 配置文件读取
- CSV / JSON 数据处理
- 定时任务
- CI/CD 中执行 PowerShell 脚本

---

## 十七、实践任务

### 任务一：基础命令练习

完成以下操作：

- 查看当前目录。
- 创建一个 `ps-lab` 目录。
- 在目录里创建 `app.log`。
- 写入几行模拟日志。
- 搜索日志中的 `error`。
- 删除实验目录。

### 任务二：服务巡检脚本

编写 `check-services.ps1`，实现：

- 接收服务名数组。
- 输出服务名称、状态、启动类型。
- 服务不存在时给出提示。
- 支持导出 CSV。

### 任务三：端口检测脚本

编写 `test-ports.ps1`，实现：

- 从数组或 CSV 读取主机和端口。
- 批量执行 `Test-NetConnection`。
- 输出连通性结果。
- 失败项单独筛选出来。

### 任务四：Windows 主机巡检脚本

编写 `check-host.ps1`，输出：

- 主机名。
- 当前用户。
- 系统版本。
- 启动时间。
- IP 地址。
- 磁盘空间。
- 监听端口。

### 任务五：远程批量巡检

准备多台 Windows 主机后，实现：

- 启用 PowerShell Remoting。
- 通过 `Invoke-Command` 批量执行巡检命令。
- 汇总结果并导出 CSV。

---

## 十八、常用命令速查

| 场景 | 命令 |
|---|---|
| 查找命令 | `Get-Command` |
| 查看帮助 | `Get-Help` |
| 查看对象属性 | `Get-Member` |
| 查看目录 | `Get-ChildItem` |
| 切换目录 | `Set-Location` |
| 查看文件 | `Get-Content` |
| 搜索文本 | `Select-String` |
| 查看进程 | `Get-Process` |
| 查看服务 | `Get-Service` |
| 筛选对象 | `Where-Object` |
| 选择属性 | `Select-Object` |
| 排序对象 | `Sort-Object` |
| 循环处理 | `ForEach-Object` |
| 导出 CSV | `Export-Csv` |
| 读取 CSV | `Import-Csv` |
| JSON 转换 | `ConvertTo-Json` / `ConvertFrom-Json` |
| 查看网络配置 | `Get-NetIPConfiguration` |
| 测试端口 | `Test-NetConnection` |
| 查看事件日志 | `Get-WinEvent` |
| 远程执行 | `Invoke-Command` |

---

## 十九、学习建议

- 学 PowerShell 时要把重点放在对象、管道、属性和脚本复用上。
- 少依赖别名，多使用完整命令，后续写脚本更清晰。
- 每学一个命令，都用 `Get-Help 命令名 -Examples` 看示例。
- 写脚本前先在命令行里一段一段验证。
- 批量修改和删除前，先输出待操作对象确认范围。
- 运维脚本尽量支持参数、日志和错误处理。
- 把常用巡检、端口检测、服务管理脚本沉淀到自己的工具目录。

---

## 二十、总结

PowerShell 的学习主线可以概括为：

```text
命令基础 -> 对象管道 -> 文件与系统管理 -> 脚本编写 -> 错误处理 -> 远程管理 -> 云平台自动化
```

如果只是日常使用，掌握文件、服务、进程、文本搜索和网络检测就已经很有帮助。

如果目标是云计算运维或 Windows 运维自动化，就需要进一步掌握脚本参数化、模块管理、远程批量执行、日志记录、CSV/JSON 处理和云平台模块。
