---
title: CRaft An Erasure-coding-supported Version of Raft for Reducing Storage Cost and Network Cost
date: 2025-10-13 19:57:27
tags:
  - 存储
  - 分布式
---

- FAST2020 主要是利用纠删码基于 Raft 进行优化，降低一致性开销

<!-- more -->

## Abstract

- 一致性协议主要是在分布式系统中用于保证可靠性和可用性的，现有的一致性协议大多都是要将日志项给备份到所有的服务器中，这种全量的副本的策略在存储和网络上的开销都很大，严重影响性能，所以后来出现了纠删码，即在保证相同的容错能力的条件下减少存储和网络的开销。
- RS-Paxos 是第一个支持 EC 数据的一致性协议，但是比起通用的一致性协议，如 Paxos/Raft，可用性都相对更差。我们指出了RSPaxos的活性问题，并试图解决，基于 Raft 提出了 CRaft，既能使用 EC 码像 RS-Paxos 一样降低存储和网络开销，也能保证如 Raft 一样的 liveness。
- 基于 CRaft 实现了一个 KVs，实验表明相对于 Raft 节省了 66% 的存储空间，写吞吐量提升了 250%，写延迟减少了 60.8%

## Introduction

- 共识算法介绍

  ：共识协议协议通常保证安全性和活动性，这意味着它们总是返回正确的结果，并且在大多数服务器都没有发生故障的情况下可以完全正常工作。

  - Google’s Chubby 会使用 Paxos 对 metadata 做副本
  - Gaios(NSDI2011) 表明一致性协议可以被用于所有数据的 replicated
  - 现如今大量应用如 etcd, TinyKV, FSS 等大规模系统都使用了 Raft/Paxos 来 replicated TB 数量级的数据，并提供更好的可用性

- **多副本介绍**：数据操作通常在分布式系统中被转换为一系列的日志指令，然后使用一致性协议在所有的服务器之间进行备份，所以数据需要经过网络传输到所有的服务器，然后还要刷会到磁盘持久化保存。一致性问题中，容错率如果为 F，那么则至少需要 N = (2F + 1) 的服务器，否则就可能因为分组的原因出现不一致的情况。因此传统的副本策略往往就意味值原始数据量的 N 倍的网络和存储开销，而且随着这些协议在大规模存储系统中得到了越来越多的应用，N 倍的网络和存储开销带来的则是延迟的增加和吞吐量的下降。**所以出现了 Erasure Coding**

- **纠删码介绍**：纠删码相比于全量拷贝的副本策略，极大地减小了存储和网络的开销。通过将数据进行分片，编码分片后的数据并生成一些校验的分片，原始的数据就能从足够数量的分片子集中恢复出来，这时候每个服务器只存储一个分片，而不是数据的全量拷贝，开销极大减小。FSS 中就使用了纠删码来减少存储开销，但是 FSS 在编码之前使用了一个 5 way 流水线 Paxos 来备份完整的用户数据和元数据，因此额外的网络开销还是有 4 倍数据量大小。

- **RS-Paxos** 是第一个结合了 Paxos 和 EC 的共识协议，虽然减少了存储和网络的开销，但是在可用性上比 Paxos 还是更差，RA-Paxos 牺牲了 liveness 来使用 EC 提升性能，换句话说就是 RS-Paoxs 如果有 N = (2F + 1) 的服务器不再能容忍 F 个错误，即容错率下降了，主要是因为 RS-Paxos 中的提交要求越来越严格。

- 作者提出了 erasure-coding-supported version of Raft **CRaft** (Coded Raft)。该方案中，一个 leader 有两种方法备份日志项到 followers，如果 leader 能够和足够数量的 followers 通信，那么 leader 将使用分片后的日志项进行备份，即传统纠删码的方式，否则将备份完整的数据以保证可用性。相比于 RS-Paxos，CRaft 最大的不同是拥有和 Paxos/Raft 相同级别的 liveness，而 RS-Paxos 没有，但是两个方案都节省了网络和存储的成本。

## Background

### Raft

- https://raft.github.io/
- Raft 原始论文：https://raft.github.io/raft.pdf
- Raft 中主要有三个角色/三种状态。Candidate 收到了来自大多数 servers 的选票后成为 Leader，一个 Server 只会给 和该 Server 日志同步的 Candidate 投票。每个 Server 每一轮最多投一次，所以 Raft 保证每一轮最多就一个 leader。
  - Leader: 处理所有客户端交互，日志复制等，一般一次只有一个Leader。
  - Follower: 类似选民，完全被动
  - Candidate候选人: 类似Proposer，可以被选为一个新的 Leader
- leader 从客户端接收日志条目，并试图将它们复制到其他服务器，迫使其他服务器的日志与自己的日志一致。当 leader 发现这一轮中有日志被被分到了大多数 servers，该日志项和之前的日志将被安全地应用到状态机中。Leader 将提交并应用这些日志项，然后告诉 followers 也 apply 他们。

![image-20251013195909977](https://raw.githubusercontent.com/zjs1224522500/PicGoImages/master//img/blog/20201216220445.png)

- 用于实际系统的共识协议通常具有以下特性：
  - Safety：它们不会在所有非拜占庭条件下返回错误的结果
  - Liveness：只要大多数服务器都处于活动状态，并且能够相互通信和与客户端通信，它们就能完全发挥作用。我们称这组服务器是健康的
- Raft 中的 Safety 是由 Leader Completeness Property 来保证的， 如果在给定 term 提交了日志条目，那么该条目将出现在所有编号较高的 term 的 leader 日志中。
- Liveness 由 Raft 规则保证，通常使用了一致性协议的系统的服务器的数量常常为奇数，假设 N = 2F + 1，Raft 可以容忍 F 个错误，我们定义一个一致性协议可以容忍的失败数量作为 liveness level，所以此时的 liveness level 为 F，更高的 liveness level 意味着更好的 liveness，没有一个协议的 liveness level 可以达到 F+1，因为如果存在这样的协议，则可能存在两个分裂的 F 个健康服务器组，这两个组可以分别就不同的内容达成一致，这是违反安全特性的。

### Erasure Coding

- 擦除编码是存储系统和网络传输中容忍错误的常用技术。们已经提出了大量的编码，其中最常用的是Reed-Solomon (RS)编码。RS 码中有两个可配置的正整数参数 k 和 m，数据被分成了相同大小的 k 个分片，然后使用这 k 个原始的数据分片计算出 m 个类似的校验分片，也就是编码过程，此时总共将有 k+m 个分片，(k,m)-RS 码就意味着所有分片中的任意 k 个分片就能恢复出原始数据，这就是 RS 码的容错原理。（类似于解方程的过程）
- 当引入一致性协议，k + m = N，N 为服务器的总数量，存储和网络开销将被见效的全拷贝的 1/k，然而如何保证 safety 和 liveness 不容忽视

### RS-Paxos

- RS-Paxos 是将纠删编码与 Paxos 相结合的一种 Paxos 的改革版本，可以节省存储和网络成本。在 Paxos 中，命令被完全传输。然而，在 RS-Paxos 中，命令是通过代码片段传输的。根据这一变化，服务器在 RS-Paxos 中只能存储和传输片段，从而降低了存储和网络成本。

- 为了保证安全性和活动性，Paxos和Raft基于以下包容-排斥原则。

  ​                                                 ∣*A*∪*B*∣=∣*A*∣+∣*B*∣−∣*A*∩*B*∣

  

包含排除原则保证在两个不同的服务器组合中至少有一个服务器的数量差距，这样安全性就可以得到保证。

- RS-Paxos 的想法是增加交集集的大小。具体来说，在选择了一个 (k,m)-RS 代码后，读quorum QR、写 quorum QW 和服务器数量 N 应该符合以下公式。

  ​													*Q**R*+*Q**W*−*N*≥*k*

___

- **本文作者：** zhengyun Li
- **版权声明：** 本博客所有文章除特别声明外，均采用[ BY-NC-SA](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。转载请注明出处！

