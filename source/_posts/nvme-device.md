---
title: crimson中的设备管理从Nvme 到Seastore
date: 2025-09-27 14:31:43
tags:
  - nvme
  - Crimson
---
NVMe 的问世，把存储硬件的性能拉到了 **“微秒 + 百万 IOPS”** 的时代。传统的内核 IO 栈和同步编程模式，反而成了新的瓶颈，无法充分释放硬件潜力。这迫使软件必须进行一次深刻的变革：从 **同步 → 异步**，从 **内核 → 用户态**，从 **粗粒度锁 → 精细化无锁并发**，才能真正发挥出 NVMe 的极限性能。

在 Ceph 的设想中，未来的存储引擎 **SeaStore** 正是为了承担这一使命：

- **同步到异步**：完全基于 Seastar 框架，采用 Future/Promise 异步编程模型，杜绝阻塞。
- **内核到用户态**：借助 Seastar 的用户态网络dpdk与 io_uring 用户态 IO 接口，最大程度减少内核切换开销。
- **绑核与分片**：每个 CPU shard 独立运行，线程与数据强绑定，端到端的数据路径天然无锁。
- **无锁化**：通过分片隔离与消息传递机制，避免传统锁竞争，让并发更细粒度、更高效。

这样一来，SeaStore 不仅是 Ceph 的“下一代引擎”，更是面向 NVMe 时代的软件范式转型的具体落地。

<!-- more -->

## 什么是NVMe

NVMe是协议，不是设备也不是驱动，就像HTTP是协议，浏览器和服务器是实现，NVMe 定义了SSD和主机控制器之间如何交流。可减少[闪存存储](https://www.ibm.com/cn-zh/topics/flash-storage)和[固态硬盘 (SSD) ]中使用的每个输入/输出 (I/O) 的系统开销。

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

## 为什么传统软件栈不够用？

虽然 Bluestore 已经绕过了文件系统（直接管理裸盘），但它仍然存在两个关键瓶颈：

1. **依赖内核 IO 栈**  
   - Bluestore 使用 libaio 提交请求，最终还是要走 Linux 内核的 I/O 路径。  
   - NVMe 的多队列并行性被内核锁与调度部分抵消。  

2. **多核扩展差**  
   - 传统ceph的OSD 内部有大量线程，多个线程共享一个设备队列，需要加锁。  
   - NVMe 给了“每个 core 一个队列”的能力，但传统 OSD 架构无法做到真正的无锁扩展。  

结果就是：  
- NVMe 单盘明明能跑 100 万 IOPS，Bluestore OSD 只能吃下 20~30 万。  
- 延迟也比硬件指标多出几倍。  

这就是 Ceph 需要 Crimson/Seastore 的根本原因。

---

## Seastar 与 NVMe 的天然契合

NVMe 的设计理念，和 Seastar 的 shard 模型几乎是天然匹配的：  

- **多队列对多 shard**  
  - NVMe：最多 64K 个提交/完成队列，每个 core 可独享一个队列。  
  - Seastar：应用拆分成多个 shard，每个 core 独立执行。  
  - 映射关系：一个 shard ↔ 一个 NVMe 队列，无需锁。  

- **无锁并行**  
  - 每个 core 只 poll 自己的完成队列，不会和其他 core 争抢。  
  - I/O 提交和完成完全 core-local，避免了传统 OSD 的“队列共享 + 自旋锁”。  

- **性能收益直观**  
  - 传统 aio + 内核队列：单盘 4KB 随机读 ~40 万 IOPS  
  - io_uring + shard 绑定队列：单盘可跑 150 万 IOPS+  
  - 在 Crimson/Seastore 的实验结果中，相比 Bluestore 延迟下降 3~5 倍，IOPS 提升 2~3 倍。  

可以说，Seastore 不是“适配 NVMe”，而是“为 NVMe 而生”。

---

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

___

