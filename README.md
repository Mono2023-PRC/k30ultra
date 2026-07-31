# K30 Ultra (cezanne) DroidSpaces + KernelSU 内核构建指南

> 为 Redmi K30 Ultra (天玑1000+, 内核4.14, MIUI 12.5) 构建支持 DroidSpaces 容器运行时的自定义内核

---

## 目录

1. [项目概述](#项目概述)
2. [前置条件](#前置条件)
3. [使用方法](#使用方法)
4. [技术细节](#技术细节)
5. [常见问题](#常见问题)
6. [参考资料](#参考资料)

---

## 项目概述

本项目提供了一套完整的 **GitHub Actions CI 工作流**，用于为 Redmi K30 Ultra 自动编译支持以下功能的自定义内核：

| 功能 | 状态 | 说明 |
|------|------|------|
| **DroidSpaces** | ✅ 完整支持 | 全命名空间隔离、OverlayFS、网络隔离 (NAT/None) |
| **KernelSU** | ✅ 支持 | 通过 kprobe 模式集成（非 GKI 设备推荐方式） |
| **path_umount** | ✅ 已 backport | 从 5.9 内核 backport，支持 KernelSU 模块卸载 |
| **Cgroups v1** | ✅ 支持 | 设备、PID、内存、调度器、 freezer、网络优先级控制器 |
| **网络隔离** | ✅ 支持 | VETH + Bridge + iptables MASQUERADE + nf_tables |

### 设备信息

| 项目 | 详情 |
|------|------|
| 设备代号 | `cezanne` |
| SoC | MediaTek Dimensity 1000+ (MT6889) |
| 内核版本 | Linux 4.14 |
| Android 版本 | Android 11 (MIUI 12.5) |
| 内核源码分支 | `cezanne-r-oss` |
| 源码地址 | [MiCode/Xiaomi_Kernel_OpenSource](https://github.com/MiCode/Xiaomi_Kernel_OpenSource) |

---

## 前置条件

### 1. 已解锁 Bootloader
你的 K30 Ultra 必须已经解锁 Bootloader。

### 2. 已安装自定义 Recovery
推荐使用 **TWRP** 或 **OrangeFox Recovery**，用于刷入编译好的内核。

### 3. GitHub 账号
你需要一个 GitHub 账号来 Fork 本项目并运行 Actions。

### 4. 备份当前内核
**强烈建议**在刷入任何自定义内核之前，先备份当前的 `boot` 分区：

```bash
adb reboot bootloader
fastboot boot twrp.img
# 在 TWRP 中备份 boot 分区
```

---

## 使用方法

### 步骤 1: 创建 GitHub 仓库

1. 登录你的 GitHub 账号
2. 创建一个新的仓库（例如 `k30ultra-droidspaces-kernel`）
3. 在仓库中创建目录 `.github/workflows/`
4. 将 `build-k30ultra-droidspaces.yml` 放入该目录

目录结构应为：
```
k30ultra-droidspaces-kernel/
└── .github/
    └── workflows/
        └── build-k30ultra-droidspaces.yml
```

### 步骤 2: 运行工作流

1. 进入你的 GitHub 仓库页面
2. 点击 **Actions** 标签
3. 在左侧选择 **"Build K30 Ultra Kernel with DroidSpaces + KernelSU"**
4. 点击 **Run workflow** 按钮
5. 配置参数：
   - **KernelSU Variant**: 选择你需要的 KernelSU 变体
     - `kernelsu` - 官方原版（推荐稳定版）
     - `kernelsu-next` - 社区维护版（功能更新）
     - `sukisu-ultra` - 另一个社区分支
   - **KernelSU Branch/Tag**: 输入版本号，例如 `v0.9.5` 或 `main`
   - **Enable SUSFS**: 不建议勾选（DroidSpaces 官方不支持 SUSFS）
6. 点击 **Run workflow**

### 步骤 3: 下载编译结果

工作流完成后：
1. 进入仓库的 **Releases** 页面，或
2. 在 Actions 运行详情页中下载 **Artifacts**
3. 下载 `K30Ultra-DroidSpaces-KernelSU.zip`

### 步骤 4: 刷入内核

1. 将手机重启到 Recovery 模式
2. 在 TWRP 中刷入下载的 `K30Ultra-DroidSpaces-KernelSU.zip`
3. 重启系统
4. 安装 KernelSU 管理器 APK
5. 安装 DroidSpaces APK
6. 打开 DroidSpaces，进入 **设置 → 需求检查 → 检查需求**

---

## 技术细节

### 1. 内核源码

使用小米官方开源的 `cezanne-r-oss` 分支：
- 仓库: https://github.com/MiCode/Xiaomi_Kernel_OpenSource
- 分支: `cezanne-r-oss`
- 该分支对应 Android R (11) 的 MTK 内核

### 2. 编译工具链

| 工具 | 版本 | 来源 |
|------|------|------|
| Clang | r416183b | AOSP Android 12 预编译工具链 |
| GCC (aarch64) | 4.9 | AOSP 预编译工具链 |
| GCC (arm) | 4.9 | AOSP 预编译工具链 |

### 3. KernelSU 集成方式（非 GKI）

由于 K30 Ultra 的内核 4.14 是非 GKI 内核，采用 **kprobe** 方式集成 KernelSU：

```
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBE_EVENTS=y
CONFIG_MODULES=y
```

KernelSU 官方说明：
> KernelSU can be integrated into non GKI kernels, and was backported to 4.14 and below.  
> If kprobe runs well in your kernel, it is recommended to use this way.  
> — [KernelSU Docs](https://kernelsu.org/guide/how-to-integrate-for-non-gki.html)

### 4. path_umount Backport

KernelSU 的 "卸载模块" 功能需要 `path_umount`，该函数在 5.9+ 内核中才有。  
工作流会自动从 5.9 内核 backport 此函数到 `fs/namespace.c`。

### 5. DroidSpaces 内核配置

根据 [DroidSpaces 官方文档](https://github.com/ravindu644/Droidspaces-OSS/blob/main/Documentation/Kernel-Configuration.md)，为 4.14 非 GKI 内核启用了以下关键配置：

**核心命名空间（必须）:**
- `CONFIG_NAMESPACES=y`
- `CONFIG_PID_NS=y`
- `CONFIG_UTS_NS=y`
- `CONFIG_IPC_NS=y`
- `CONFIG_USER_NS=y`

**Cgroups 支持:**
- `CONFIG_CGROUPS=y`
- `CONFIG_CGROUP_DEVICE=y`
- `CONFIG_CGROUP_PIDS=y`
- `CONFIG_MEMCG=y`
- `CONFIG_CGROUP_SCHED=y`

**网络隔离（NAT/None 模式）:**
- `CONFIG_NET_NS=y`
- `CONFIG_VETH=y`
- `CONFIG_BRIDGE=y`
- `CONFIG_NF_NAT=y`
- `CONFIG_IP_NF_TARGET_MASQUERADE=y`
- `CONFIG_NETFILTER_XT_TARGET_MASQUERADE=y`

**文件系统支持:**
- `CONFIG_OVERLAY_FS=y` (volatile 模式必需)
- `CONFIG_DEVTMPFS=y` (容器 /dev 必需)
- `CONFIG_BLK_DEV_LOOP=y`

**关键兼容性设置:**
- `CONFIG_ANDROID_PARANOID_NETWORK=n` (旧内核需要关闭以支持容器网络)

### 6. 已知问题与注意事项

| 问题 | 说明 | 解决方案 |
|------|------|----------|
| **VFS Deadlock (4.14.113)** | 某些 4.14 内核版本存在 `grab_super()` 死锁，导致 systemd 挂起 | 在 DroidSpaces 应用中启用 **Deadlock Shield**，或使用 Alpine Linux |
| **Nested Containers** | Docker/LXC 在 4.14 上可能遇到 BPF 限制 | 配置 Docker 使用 `cgroupfs` driver 和 `vfs` storage driver |
| **nftables 兼容性** | 4.14 缺乏完整的 nftables 支持 | 在容器内使用 `iptables-legacy` 替代 |
| **SUSFS 冲突** | DroidSpaces 官方不支持 SUSFS | 如果必须使用，关闭 "HIDE SUS MOUNTS FOR ALL PROCESSES" |

---

## 常见问题

### Q1: 编译失败怎么办？

1. 检查 GitHub Actions 日志，查看具体错误
2. 常见原因：
   - 工具链下载失败（网络问题）→ 重试
   - defconfig 名称错误 → 确认 `cezanne_defconfig` 存在
   - 内存不足 → GitHub Actions 免费版有 7GB 内存限制，通常足够编译内核

### Q2: 刷入后无法开机？

1. 进入 Recovery，刷回之前备份的 boot 镜像
2. 检查是否使用了正确的 defconfig
3. 尝试禁用 `CONFIG_ANDROID_PARANOID_NETWORK=n`（某些 ROM 需要它）

### Q3: DroidSpaces 需求检查失败？

在终端运行：
```bash
su -c droidspaces check
```

查看具体缺少哪些功能，然后检查内核配置是否包含对应选项。

### Q4: 网络隔离模式不工作？

确保：
- `CONFIG_NET_NS=y`
- `CONFIG_VETH=y`
- `CONFIG_BRIDGE=y`
- `CONFIG_ANDROID_PARANOID_NETWORK=n`
- 容器内使用 `iptables-legacy`（4.14 不支持完整 nftables）

---

## 参考资料

| 资源 | 链接 |
|------|------|
| DroidSpaces 官方仓库 | https://github.com/ravindu644/Droidspaces-OSS |
| DroidSpaces 内核配置文档 | https://github.com/ravindu644/Droidspaces-OSS/blob/main/Documentation/Kernel-Configuration.md |
| KernelSU 官方文档 (非 GKI) | https://kernelsu.org/guide/how-to-integrate-for-non-gki.html |
| 小米内核开源仓库 | https://github.com/MiCode/Xiaomi_Kernel_OpenSource |
| AnyKernel3 | https://github.com/osm0sis/AnyKernel3 |
| OnePlus 7 Pro DroidSpaces 内核参考 | https://github.com/charan-gn/guacamole-droidspaces-kernel |
| Redmi Note 8 DroidSpaces 内核参考 | https://github.com/reygasta/kernel-ginkgo-droidspaces |

---

## 免责声明

⚠️ **刷机有风险，操作需谨慎！**

- 本工作流仅供学习和研究使用
- 编译和刷入自定义内核可能导致设备变砖、数据丢失或失去保修
- 请在操作前备份重要数据
- 作者不对因使用本项目导致的任何损失负责

---

*Generated for Redmi K30 Ultra (cezanne) - Kernel 4.14 - Android 11*
