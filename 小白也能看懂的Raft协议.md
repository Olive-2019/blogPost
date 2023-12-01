---
title: 小白也能看懂的Raft协议
date: 2023-11-30 22:49:41
tags:
- Raft
- 容错
- 分布式
---
如题

<!-- more -->

# Raft是什么？

专业术语(~~高大上的话~~)：分布式共识协议，确保在分布式系统中各个节点之间达成一致的数据状态。

大白话：让所有服务器（节点）统一状态

# 大概是怎么做到的？

核心思想：Raft算法通过**选举Leader**，让Leader确保系统中的所有节点以一致的顺序接收和**应用**相同的指令序列，从而保证数据一致性和可靠性。

Raft将共识问题分成两个子问题：

1. Leader选举
2. 日志同步

此外，我们还有两个重要的假设：

1. term： 系统时间片。提供follower超时选举、candidate重新选举、防止brain split等功能。每次开启选举时+1（包括Follower超时选举和Candidate重新选举）

   > brain split：脑裂，即指系统中同时出现多个leader的情况
   >
2. RPC：所有的数据都是从Leader/Candidate流向Follower的，Leader往外发的**AppendEntries**提供日志复制和心跳功能，Candidate往外发的**RequestVote**提供请求选票功能

## 选举

### 系统状态

Raft系统中定义了三个不同的状态，每个节点仅为其中一个状态。

1. Follower：完全被动的接收来自其他两种server的指令
2. Candidate：选举过程中的中间状态
3. Leader：保证系统中其他节点和本节点的同步

### 选举流程：

1. **超时选举**：节点进入系统是follower状态，如果一段随机时间内都没有收到Leader发送的AppendEntries，则term自增，超时选举成为Candidate。
2. **候选**：Candidate选举过程中若收到过半选票则成为Leader，若收到所有回应，但选票没过半，则term自增，重新选举
投票原则：每个term只投一次，只投给term比自己大且日志比自己新的节点（依次比较term和index）
3. **Leader的初始化工作**：Leader的初始化操作包括：发送心跳维护地位，初始化next和applied，前者为日志条数，后者为-1

> 与论文不同，本系统日志index从0开始计数

4. **状态回退与更新**：Candidate和Leader运行过程中若收到term比自己大的指令，则成为Follower，并更新term；Follower若收到term比自己大的指令，则只更新term

## 日志同步
### 日志内容
1. index：每个日志条目都有一个唯一的索引值，用于标识该条目在日志中的位置。
2. term：任期号，表示该条目所处的任期（Leader接收该命令的任期）
3. command：需要在状态机上执行的指令
### 日志复制
核心思想是Leader将Follower的日志同步到自己的状态序列。具体做法是Leader利用AppendEntries同步。
1. Leader的next数组和applied数组分别记录要向每个follower发送的下一条日志和已经确认的日志。前者用于发送日志，初始化值为日志的末端；后者用于确认commit，初始化值为日志前端。
2. Leader发送AppendEntries时，会带上prevIndex和prevTerm，表示上一条日志的index和term。保证Follower能衔接上，若被拒绝，则nextIndex自减再发
3. Follower若在自己的日志序列中能找到与prevIndex和prevTerm匹配的日志，则将日志复制到自己的日志序列中，并确认该日志，否则拒绝
# 具体实现逻辑
## Leader
### 主线程
1. 初始化nextIndex和appliedIndex
2. 发送心跳
3. 检测follower返回值并发送新的心跳/日志
### 接收AppendEntries线程
1. 检测term，若比自己大则退回到follower状态并以follower状态
### 接收RequestVote线程
# Raft应用场景

# Raft优化
