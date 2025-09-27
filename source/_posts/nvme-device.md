---
title: crimson中的设备管理从Nvme 到Seastore
date: 2025-09-27 14:31:43
tags:
  - nvme
  - Crimson
---
现在的存储速度，传统 SATA SSD 已经跟不上了。于是 NVMe 出场了——更快、更并行、延迟低。Ceph 的 Crimson 正是为了迎接这样的下一代存储而生，它用 Seastar 来管理这些 NVMe 设备，把每个设备、每个队列都当作可调度的资源，让多核 CPU 的性能被充分发挥。今天，我们就从 NVMe 设备聊起，顺着 Crimson 的设备管理一路看下去，看看底层到底是怎么跑的。

<!-- more -->
## 什么是NVMe

NVMe是协议，不是设备也不是驱动，就像HTTP是协议，浏览器和服务器是实现，NVMe 定义了SSD和主机控制器之间如何交流。

NVMe设计的初衷是针对**非易失性存储介质（NAND、3D XPoint、持久内存等）** 设计。

不再兼容HDD时代的AHCI设计限制（单队列、深度32）

关键特征：

+ 队列模型

  + 支持64K个提交队列/完成队列，每个队列深度64k.
  + 每个cpu核可以有独立队列，减少延迟，充分释放闪存的并行性。

+ 寄存器模型优化

  + 一条IO命令只需要写一个doorbell寄存器，而AHCI需要四次寄存器读写

    AHCI模型（SATA SSD/HDD 用的传统协议），AHCI定义了一个命令列表，在内存里。

    当你要下发一条I/O命令（比如读4KB数据）：

    1. CPU 把命令描述符写入内存里的 Command List。
    2. CPU 必须更新多个寄存器：写 **命令头寄存器** 写 **命令表基址寄存器**写 **PRDT（物理区域描述表）寄存器**更新 **控制寄存器** 告诉控制器有新命令,控制器才能去取命令，开始执行
    3. 每个 I/O 至少 **4 次 MMIO（寄存器读写）开销**，消耗大量 CPU cycle（大概 2000~8000 个 cycle），延迟在微秒级。

    NVMe模型，定义了提交队列（SQ）/ 完成队列（CQ）,放在主机内存中。

    下发一条I/O命令的步骤：

    1. CPU 把命令写到内存里的 **SQ 队列槽位**。
    2. CPU 只需要 **写一次 doorbell 寄存器**，告诉控制器“SQ head 已经增加”。
    3. 控制器会直接 DMA 去取命令，不需要 CPU 做额外寄存器交互。
    4. 每个 I/O 只有 **1 次 MMIO（doorbell write）**，极大减少 CPU 开销和延迟。

+ 延迟极低

  + 单次 I/O 延迟 ~10µs 级别，比 AHCI 少一半以上。

+ 协议分层清晰：

  + PCIe 提供物理链路
  + NVMe协议再PCIe之上定义逻辑指令集

## **设备特性**

+ 对齐要求

  + **最小 I/O 单位**：NVMe SSD 通常要求 **4KB 对齐**，即提交的 I/O 地址和长度最好是 4KB 的整数倍。
  + **擦除单元 (Erase Block / LBA Block)**：
    - NAND 闪存以 Block 为单位擦写。
    - 随机写小于 Block 会触发 **读-改-写操作**，造成 **写放大**（Write Amplification），降低寿命和性能

+ 写放大

  + 为了更新少量数据，SSD 需要擦除和重写整个 Block，导致实际写入数据量 > 应用写入数据量

  影响：小随机写 IOPS 下降。设备寿命减少。

  优化方式：使用 **顺序写** 或 **批量写** 、合理规划 **文件系统块大小** 与 NVMe 对齐。

+ 延迟指标

  + 单次 I/O 延迟通常：**消费级 NVMe**：读 10–30 µs，写 20–50 µs。**企业级 NVMe**：读/写可 < 10 µs。
  + 对比SATA SSD：读/写50–100 µs  HDD :读/写 5–10 ms

+ iops与吞吐上限

  + **单盘 IOPS**：随机读(4KB) 500k-3000k .随机写（4kB）300k-700k
  + NVMe 原生支持多队列，每个 CPU 核心可绑定一个或多个提交队列（这里就体现出crimsn 中使用seastar框架的重要性了）
  + 在多核 CPU 和多队列的场景下，IOPS 可以线性扩展到数百万，充分利用硬件并行能力。

+ I/O调用方式

  + libaio(传统linux异步io,适合单线程或者少量队列异步提交)
  + io_uring（linux5.1 真正零拷贝和高性能多队列异步I/O）

+ 高级特性：

  + NVMe2.0 :在线升级、多命名空间（128个）、多流、原子写、 sanitize、copy、verify、compare等高级特性。
  + 遵循OCP 2.5规范协议，Telemetry、Latency Monitor、Thermal Throttle等高级特性
  + 支持SR-IOV • 最多64个V
  + 灵活数据放置(FDP) • 提升稳态随机写性能，大幅降低写放大，特殊场景下 可实现WAF=1
  + TCG OPAL2.0安全规范 • 支持OPAL 2.0协议栈 • 支持SM2/SM3/SM4/AES256数据加密
  + 端到端保护
  + 持Secure Boot、Firmware安全校验、Format、 Sanitize等多种企业级安全特性



简单了解了NVMe 协议和设备的特性，

## 2️⃣ 软件工程层（开发 Crimson/Seastore 必须掌握）

- **I/O 路径**
  - 内核 NVMe 驱动怎么和 `io_uring` 打通。
  - 为什么用户态驱动（SPDK）能进一步降低延迟。
  - O_DIRECT、DMA 映射对性能的影响。
- **多核并行**
  - NVMe 支持多队列，你的软件要怎么分配（比如 Seastar shard 对应 NVMe queue pair）。
  - 如何避免多个 core 抢一个 queue 带来的锁竞争。
- **错误处理**
  - `EINVAL (Invalid argument)`：未对齐 I/O。
  - `EIO`：设备层报错，需要 retry。
  - reset / requeue 的语义（设备热插拔时必须考虑）。
- **性能优化**
  - 顺序写 > 随机写，如何利用 append-log 结构减少随机写。
  - 大块 I/O（128K、1M）和小块 I/O 的 trade-off。
  - IOPS vs 吞吐量的权衡。

## 3️⃣ 深入机制层（进阶，写存储引擎/GC/调度必须掌握）

- **Flash 物理特性**
  - Page / Block / Plane 的层次结构。
  - 为什么必须“先擦后写”。
  - 写放大（WA）和 GC 的触发机制。
  - 耐久性（DWPD：Drive Writes Per Day）。
- **NVMe 协议细节**
  - NVMe 命令集（Read, Write, Flush, Dataset Management, Zone Append）。
  - Zoned Namespace (ZNS) SSD 的新模式（只能顺序写 Zone）。
  - NVMe 2.0 里的 KV SSD 扩展（key-value 命令）。
- **持久性保证**
  - Flush/FUA 的语义（写入必须持久化到 NAND）。
  - Ceph Crimson 里 journal write + flush 的正确用法。
  - Power-loss protection（掉电保护）的区别。

## 4️⃣ 前沿与趋势层（存储软件研发 leader 需要了解）

- **SPDK / DPDK + NVMe-oF**
  - 用户态驱动模型，绕过内核栈。
  - 远程存储：NVMe over Fabrics（RoCE、TCP）。
- **ZNS SSD 与软件定义存储**
  - 用软件控制写放大和 GC，替代 FTL（Flash Translation Layer）。
- **CXL 与持久内存**
  - NVMe 未来可能和 CXL 竞争/融合。

## 为什么要关注设备管理？

## Crimson 的设备抽象？

## 设备初始化流程？

## 关键问题？

