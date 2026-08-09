# ArchLinux VM (ansible-ctl) 优化记录

**日期**：2026-08-09
**VM 路径**：`D:\Virtual Machines\ArchLinux\`
**参考文档**：[ansible-vmware-lab.md](ansible-vmware-lab.md)

## 修改概览

| # | 配置项 | 修改前 | 修改后 | 原因 |
|---|--------|--------|--------|------|
| 1 | `ethernet0.virtualDev` | `e1000` | `vmxnet3` | 半虚拟化网卡，CPU 开销更低，吞吐更高 |
| 2 | `guestOS` | `other6xlinux-64` | `archlinux-64` | 让 VMware 针对 Arch Linux 应用正确的优化策略 |
| 3 | `cpuid.coresPerSocket` | `1` | `2` | 单 socket 多核拓扑，NUMA 调度更优 |
| 4 | `sata0:1.present` | `TRUE` | `FALSE` | 安装完成后卸载 ISO，避免意外启动到安装镜像 |
| 5 | `tools.syncTime` | `FALSE` | `TRUE` | 开启宿主机-客户机时间同步，保证 SSH/Ansible 时间戳一致 |
| 6 | `sound.present` | `TRUE` | `FALSE` | 服务器/控制节点无需声卡，释放资源 |
| 7 | `svga.vramSize` | `268435456` (256MB) | `16777216` (16MB) | 控制台文本操作为主，256MB 显存浪费 |
| 8 | `mem.hotadd` | `TRUE` | `FALSE` | 实验环境无需热添加，4GB 已固定 |
| 9 | `vcpu.hotadd` | `TRUE` | `FALSE` | 同上，2 vCPU 已固定 |

## 修改详情

### 1. 网卡：e1000 → vmxnet3

```
- ethernet0.virtualDev = "e1000"
+ ethernet0.virtualDev = "vmxnet3"
```

`e1000` 模拟 Intel 82545EM 千兆网卡，每个数据包都需要 VM 内核和 VMware 模拟层之间多次上下文切换。`vmxnet3` 是 VMware 的半虚拟化（paravirtualized）驱动，Arch Linux 内核已内置 `vmxnet3` 模块，无需额外安装。

对 Ansible 控制节点而言，所有 SSH 连接、模块传输、facts 收集都走网络，网卡性能直接影响操作体验。

### 2. 客户机操作系统类型：other6xlinux-64 → archlinux-64

```
- guestOS = "other6xlinux-64"
+ guestOS = "archlinux-64"
```

VMware 根据 `guestOS` 决定：
- 默认内存管理策略（ballooning、swap）
- I/O 调度器模拟行为
- ACPI / 电源管理表
- 推荐的虚拟硬件类型

`other6xlinux-64` 是兜底配置，VMware 不会做任何针对性优化。`archlinux-64` 让 VMware 识别为滚动发行版 64 位 Linux，应用更合理的默认值。

### 3. CPU 拓扑：2 socket × 1 core → 1 socket × 2 cores

```
- cpuid.coresPerSocket = "1"
+ cpuid.coresPerSocket = "2"
```

最终效果：`numvcpus = "2"` + `cpuid.coresPerSocket = "2"` → 1 socket × 2 cores。
- 减少跨 socket 的 NUMA 开销（虽然单宿主机无 NUMA，但客户机内核仍按 socket 拓扑调度）
- 某些软件（数据库、JVM）按 socket 授权，单 socket 不会触发限制

### 4. 卸载安装 ISO

```
- sata0:1.present = "TRUE"
+ sata0:1.present = "FALSE"
```

Arch Linux 已安装完成运行中，保留 ISO 挂载没有意义，且重启时若 BIOS 启动顺序将 CD-ROM 排在前可能误入安装环境。

### 5. 时间同步

```
- tools.syncTime = "FALSE"
+ tools.syncTime = "TRUE"
```

VMware Tools 周期性将客户机时钟与宿主机同步。Ansible 依赖一致的时间戳，且 SSH 密钥认证在时钟偏差过大时可能失败。文档也明确要求「节点时间同步正常」。

配合 VM 内部安装 `open-vm-tools`：
```bash
sudo pacman -S --needed open-vm-tools
sudo systemctl enable --now vmtoolsd
```

### 6. 禁用声卡

```
- sound.present = "TRUE"
+ sound.present = "FALSE"
```

Ansible 控制节点通过 SSH 终端操作，无需音频设备。禁用声卡减少虚拟硬件数量和中断开销。

### 7. 显存：256MB → 16MB

```
- svga.vramSize = "268435456"
+ svga.vramSize = "16777216"
```

控制台操作为主，不需要图形加速。16MB 足够 X11/Wayland 基础桌面操作（如需要），远低于原来的 256MB。释放的宿主机内存可分配给其他 VM。

### 8-9. 禁用热添加

```
- mem.hotadd = "TRUE"
- vcpu.hotadd = "TRUE"
+ mem.hotadd = "FALSE"
+ vcpu.hotadd = "FALSE"
```

热添加在本实验环境中没有实用价值——5 台 VM 的配置是固定的。启用热添加会迫使客户机内核加载额外的 ACPI 表和驱动，增加启动时间和内存占用。

## 生效方式

**这些修改需要 VM 关机后重新开机才能生效**（热修改部分不适用）。建议操作：

1. 在 VM 内执行 `sudo poweroff`
2. 确认 `.lck` 目录和 `.vmem` 文件已消失
3. 在 VMware 中重新启动 VM

## VM 内部还需确认

修改 `.vmx` 后，VM 内部还需做以下验证：

```bash
# 1. 确认网卡变为 vmxnet3
lspci | grep -i ethernet
# 预期：VMware VMXNET3 Ethernet Controller

# 2. 确认 open-vm-tools 已安装
pacman -Q open-vm-tools
# 若未安装：sudo pacman -S --needed open-vm-tools

# 3. 确认时间同步已启用
vmware-toolbox-cmd timesync status
# 或
systemctl status vmtoolsd

# 4. 确认静态 IP 为 192.168.88.10
ip -brief address
```