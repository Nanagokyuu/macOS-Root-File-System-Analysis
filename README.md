# macOS Golden Gate (Apple Silicon) 根文件系统、APFS Volume Group 与启动链分析

**文档版本：** v1.0  
**分析对象：** MacBook Pro 14（macOS Golden Gate 27.0 beta 1）  
**分析方式：** 实机验证（diskutil、mount、firmlink、snapshot、APFS Volume Group、Simulator Runtime）

---

# 1. 概述

从 macOS Catalina 开始，Apple 将系统拆分为：

* System Volume（系统卷）
* Data Volume（数据卷）

从 Big Sur 开始进一步引入：

* Signed System Volume (SSV)
* APFS Snapshot Boot
* Firmlink Namespace

而在 Apple Silicon 上，又增加：

* iSCPreboot
* xART
* Hardware
* Secure Boot Policy

因此现代 macOS 的根目录 `/` 已不再对应某一个真实卷，而是多个卷共同构建出的统一命名空间（Namespace Union）。

---

# 2. APFS 容器结构

本机主要 APFS 容器：

```text
disk3
├── disk3s1  System
├── disk3s2  Preboot
├── disk3s3  Recovery
├── disk3s5  Data
└── disk3s6  VM
```

其中：

```text
disk3s1
Name: Macintosh HD
Role: System

disk3s5
Name: Data
Role: Data
```

属于同一个：

```text
APFS Volume Group
UUID:
937449E6-EFC1-4FF8-B7B9-10B984769757
```

---

# 3. 实际启动卷

系统实际并非从：

```text
disk3s1
```

启动。

验证：

```bash
mount | grep ' / '
```

结果：

```text
/dev/disk3s1s1 on /
(apfs,sealed,read-only)
```

说明：

```text
disk3s1
    ↓
Snapshot
    ↓
disk3s1s1
    ↓
挂载到 /
```

结构：

```text
disk3s1
(System Volume)
    │
    ▼
Signed Snapshot
(disk3s1s1)
    │
    ▼
/
```

---

# 4. Signed System Volume（SSV）

验证：

```bash
diskutil info /
```

结果：

```text
Sealed: Yes
Volume Read-Only: Yes
```

说明：

当前启动的是：

```text
Signed System Volume
```

特点：

* 只读
* Apple 签名
* Hash Tree 校验
* Secure Boot 验证

因此：

```text
/System
/bin
/sbin
/usr
```

均来自：

```text
disk3s1s1
```

而非 Data 卷。

---

# 5. Root Namespace Union

验证：

```bash
ls -ldO /
```

结果：

```text
sunlnk
```

同时：

```bash
ls -ldO /System/Volumes/Data
```

结果：

```text
sunlnk
```

说明：

系统使用：

```text
Namespace Union
```

机制。

逻辑结构：

```text
System Snapshot
        +
Data Volume
        │
        ▼
统一根目录 /
```

用户看到的是一个合并后的视图。

---

# 6. Firmlink 机制

验证：

```bash
cat /usr/share/firmlinks
```

结果：

```text
/Applications
/Users
/Library
/private
/opt
/cores
/pkg
/usr/local
...
```

这些目录实际上位于：

```text
/System/Volumes/Data
```

例如：

```text
/Applications
        │
        ▼
/System/Volumes/Data/Applications
```

```text
/Users
        │
        ▼
/System/Volumes/Data/Users
```

```text
/Library
        │
        ▼
/System/Volumes/Data/Library
```

Firmlink 与 Symbolic Link 不同：

| 特性        | Firmlink | Symlink |
| --------- | -------- | ------- |
| 内核级       | 是        | 否       |
| 用户可见      | 否        | 是       |
| 跨卷映射      | 是        | 否       |
| Finder 显示 | 普通目录     | 链接      |

---

# 7. 根目录结构

## 7.1 Data Volume 实际存储目录

```text
/Applications
/Users
/Library
/private
/opt
/cores
/pkg
```

实际位于：

```text
/System/Volumes/Data
```

---

## 7.2 Symbolic Link

### etc

```text
/etc
    ↓
private/etc
```

---

### tmp

```text
/tmp
    ↓
private/tmp
```

---

### var

```text
/var
    ↓
private/var
```

---

### home

验证：

```bash
mount | grep home
```

结果：

```text
auto_home
```

结构：

```text
/home
    ↓
/System/Volumes/Data/home
```

实际由：

```text
autofs
```

动态管理。

---

# 8. Volumes 目录

验证：

```bash
ls -ld /Volumes/"Macintosh HD"
```

结果：

```text
Macintosh HD -> /
```

因此：

```text
/Volumes/Macintosh HD
```

实际上是：

```text
符号链接
```

指向：

```text
/
```

形成：

```text
/Volumes/Macintosh HD
      │
      ▼
      /
```

用于兼容旧软件和 Finder 导航逻辑。

---

# 9. Data Volume

挂载：

```text
disk3s5
```

路径：

```text
/System/Volumes/Data
```

验证：

```bash
mount
```

结果：

```text
protect
root data
```

说明：

该卷是：

```text
Root Data Volume
```

作用：

* 用户数据
* 第三方应用
* Home Directory
* 可写系统数据

---

# 10. Preboot Volume

卷：

```text
disk3s2
```

挂载：

```text
/System/Volumes/Preboot
```

作用：

```text
Kernel Collections
Boot Manifest
Boot Support Files
```

用于启动阶段加载系统。

---

# 11. Recovery Volume

卷：

```text
disk3s3
```

Role：

```text
Recovery
```

平时不挂载。

用于：

```text
macOS Recovery
恢复系统
磁盘工具
终端
重新安装系统
```

---

# 12. VM Volume

卷：

```text
disk3s6
```

挂载：

```text
/System/Volumes/VM
```

作用：

```text
swapfile
memory pressure
虚拟内存
```

挂载参数：

```text
noexec
```

禁止执行文件。

---

# 13. Update Volume

卷：

```text
disk3s4
```

挂载：

```text
/System/Volumes/Update
```

作用：

```text
系统更新缓存
SSV 构建
更新中间状态
```

---

# 14. Apple Silicon 辅助启动容器

Apple Silicon 额外存在：

```text
disk1
```

结构：

```text
disk1
├── iSCPreboot
├── xART
├── Hardware
└── Recovery
```

---

# 15. iSCPreboot

卷：

```text
disk1s1
```

挂载：

```text
/System/Volumes/iSCPreboot
```

发现：

```text
937449E6-EFC1...
```

目录名与：

```text
APFS Volume Group UUID
```

完全一致。

内部：

```text
LocalPolicy/
```

因此可确认：

```text
iSCPreboot
```

用于：

```text
Apple Silicon Boot Policy
```

包括：

* Startup Security
* Reduced Security
* External Boot Policy
* LocalPolicy

结构：

```text
iSCPreboot
├── VolumeGroupUUID
│   └── LocalPolicy
├── SystemRecovery
├── WiFi
└── SFR
```

---

# 16. xART

卷：

```text
disk1s2
```

挂载：

```text
/System/Volumes/xarts
```

内容：

```text
A375F60A-....gl
```

特点：

* 大小约 6 MB
* SIP 保护
* root 无法读取

推断用途：

```text
Trust Cache
Boot Artifacts
Secure Boot Metadata
IMG4 / IM4P 相关数据
```

Apple 未公开详细文档。

---

# 17. Hardware Volume

卷：

```text
disk1s3
```

挂载：

```text
/System/Volumes/Hardware
```

发现：

```text
FactoryData
MobileActivation
ProductDocuments
dramecc
recoverylogd
srvo
```

用途：

```text
设备激活信息
工厂数据
硬件状态记录
ECC 记录
Recovery 日志
```

属于 Apple Silicon 启动辅助数据卷。

---

# 18. Simulator Runtime

本机存在两套 Simulator Runtime 机制。

## 18.1 传统 Runtime

```text
disk5s1
```

挂载：

```text
/Library/Developer/CoreSimulator/Volumes/iOS_23E254a
```

对应：

```text
iOS 26.4.1 Simulator
```

---

## 18.2 Cryptex Runtime

```text
disk7s1
```

挂载：

```text
/private/var/run/com.apple.security.cryptexd/mnt/...
```

对应：

```text
iOS 27.0 Simulator
```

说明：

新版 Simulator 已开始使用：

```text
Cryptex
```

机制动态挂载运行时镜像。

---

# 19. APFS 加密用户

验证：

```bash
diskutil apfs listUsers /
```

结果：

```text
Local Open Directory User
Volume Owner: Yes

Personal Recovery User
Volume Owner: Yes
```

说明：

APFS 用户并不等于 macOS 登录用户。

其中：

```text
Personal Recovery User
```

用于：

* FileVault 解锁
* Personal Recovery Key
* Recovery 环境认证

---

# 20. 完整启动链

```text
Boot ROM
    │
    ▼
LLB
    │
    ▼
iBoot
    │
    ▼
iSCPreboot
    │
    └── LocalPolicy
            (启动策略)
    │
    ▼
xART
    │
    └── Trust Cache
        Boot Artifacts
    │
    ▼
disk3s1
(System Volume)
    │
    ▼
Signed Snapshot
(disk3s1s1)
    │
    ▼
Namespace Union
    │
    ├── System Snapshot
    └── Data Volume
    │
    ▼
最终根目录 /
```

---

# 21. 最终结论

现代 Apple Silicon macOS 的 `/` 并不是单一文件系统，而是由以下组件共同构建：

```text
Boot Policy
(iSCPreboot)
        +
Secure Boot Artifacts
(xART)
        +
Signed System Snapshot
(disk3s1s1)
        +
Root Data Volume
(disk3s5)
        +
Firmlink Namespace
        +
Symlink Layer
        +
Autofs
        │
        ▼
最终用户看到的 /
```

从系统实现角度来看，macOS 根目录实际上是一个由 **APFS Snapshot、Volume Group、Firmlink、Secure Boot 和 Namespace Union** 共同组成的逻辑文件系统视图，而非传统意义上的单一磁盘分区。
