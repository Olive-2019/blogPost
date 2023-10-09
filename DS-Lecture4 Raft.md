---
title: 分布式系统 Raft
date: 2023-10-05 15:03:37
tags:
    - mit 6.824
    - 容错
    - Raft
---
分布式协同算法 Raft

<!-- more -->

# 论文阅读

以前的算法存在两个问题，1. 不容易理解2.对于现实问题没有好的基础，所以raft设计的目标就是好理解。为此，Raft使用了两个方法，1.问题分解2.简化系统状态
raft算法核心是leader，leader的产生简化了系统，因为leader可以决定新日志条目存放的地方，数据流也只从leader流向其他server。系统最简单来说就是两步

1. 选举leader
2. leader管理复制日志

具体而言，Raft将问题分解为以下几个子问题

1. **leader选举**
2. **日志复制**：leader管理
3. **安全**：保证系统属性安全

## Raft基础

### 状态state

系统中的server有三个状态 `leader`、`follower`、`candidate`。
`follower`是完全被动的接收来自其他两种server的指令；`leader`会处理所有来自client的请求，其他server也将请求转发给 `leader`；`candidate`则是选举过程中的中间状态。

```mermaid
graph TD
    A[Follower] -->|开始选举| B(Candidate)
    B --> |重新选举|B
    B --> |获得大多数选票| C(Leader)
    C --> |其他server有更大的term编号| A
    B --> |当前系统有leader了or其他server有更大的term编号| A
```

### 时间片term

系统的term（时间片）是任意(随机)长度的，每一个term都由选举开始，如果选举失败，该term立即终止，马上进入下一个term。Raft始终保证系统中有且仅有一个leader。
系统中的server保存了当前的term编号，如果server发现自己的term编号比其他server小，则自增到新的term编号，如果leader或candidate发现自己的编号过时了，则马上降级到follower
如果server收到了term编号不对的请求，将该请求丢弃。

### 通信 RPC

系统中需要三条RPC通信路径

1. **RequestVote RPCs**：在选举过程中，由候选者初始化
2. **AppendEntries RPCs**：由leader初始化，在系统运行时提供日志条目复制和心跳功能
3. **TransferSnapshot RPCs**：传递server快照的RPCs

## Raft性质

Raft算法运行中全程保证这些性质成立

1. **选举安全**：一个term中至多存在一个leader
2. **leader仅追加log**：leader的log不会被其他server覆写
3. **log匹配**：两个拥有相同index和term的log必然完全相同，且前序log也相同
4. **leader完整性**：committed的log entry一定会在后续的leader log中出现
5. **状态机安全性**：

## 选举

server以follower身份启动，leader会定时发心跳信息（无日志信息的appendEntries）给follower维护自己的leader地位。当follower超时未收到心跳信息时则开启选举。
选举过程是follower将term编号自增并进入candidate状态，然后给自己投票，并向其他server并行发出RequestVote RPCs请求票。
每个server在一个term里至多给一个server投票，投票遵循先到先得原则。

### 选举结果

candidates状态的结果有三种：

1. 获取大部分选票，赢得选举->leader 开始发心跳信息维护自己的leader地位
2. 收到了其他server的RequestVote RPCs，若其term编号大于等于自己，则认定其为合法leader->follower （否则自己仍为candidate）
3. 当多个server同时进入candidate状态并发送RequestVote RPCs，没人赢得选举->candidate 重启选举流程

### 随机超时机制

为了保证系统尽量少进入第三个状态，即无人赢得选举，不停地选举，Raft采用随机的超时时限（150-300ms之间）。这样可以保证在在某一个时刻，只有一个server发现超时，并赢得选举。
每一个candidate在term开始时重新获取一个随机超时时限，

## 日志复制

当client发送命令时，系统中会按顺序发生如下事件：

1. leader追加一个新的log entry，也就是这条命令
2. 发布AppendEntries RPCs给其他server
3. 当这个entry完成安全的复制后，leader会执行该命令
4. 返回执行结果给client

### 日志内容

日志将包含两个部分：

1. 状态机命令
2. term编号：标记这条命令产生的时期

### commited entry

* 当entry被大多数（为什么是大多数？）server复制时，该log entry会变成commited状态。Raft将会保证系统中commited的entry被持久化以及被所有server执行。
* 当一条log entry是commited时，这条日志之前的所有entries都会被commit，不论任何leader产生的。
* leader会追踪将要commit的最早entry，并用其发送appendEntries RPCs
* 当follower得知entry被commit以后，就会执行该entry包含的命令

### 运行时日志一致性保证

日志有两个属性

* index和term相同的log，命令必然相同
* index和term相同的log，前置log必然相同

发送AppendEntries RPC时，leader会将前一个log的index和term也附上，如果follower没找到前一个log，那也不读入当前log

### leader对日志一致性保证

正常运行时，leader和多个follower的日志将会保持一致，当leader挂掉重新选举时，可能会导致日志出现不一致的现象，此时leader将自己的log强制复制给follower。leader会为**每个follower**更新nextIndex，用来指示向特定follower发送的下一条log entry。具体做法是：

1. **初始化**：将nextIndex初始化为leader的最后一条log entry的下一条
2. **试错**：给follower发送下一条的AppendEntries，如果失败，nextIndex--（也可以通过返回entries，让leader直接查找nextIndex）
3. **结果**：当AppendEntries成功时，正确的nextIndex就找到了
4. **更新**：将leader后续的log entries通过AppendEntries发送给follower，覆盖掉不一致的日志

通过这些措施，leader在选举成功后，就会让整个系统里的follower逐渐收敛到自己的日志中来。同时，leader是不会删除（覆盖）自己的日志，只会覆盖掉follower的日志

## 安全性

### 选举限制
为了简化算法，只允许log entries从leader流向follower，Raft保证leader会包含已经被提交的所有log entries。
Raft使用voting的方法来保证leader拥有所有的log entries。candidate发送带有自己所有log entries的RequestVote RPC给其他server，如果其他server发现candidate的log entries没有自己新，则否决该candidate。
判断log entries新旧方法：比较最后一个log entry的index和term，term编号更大的更新，如果term编号一样，则index更大的更新

### 提交限制
leader只能提交当前term中的log entry，相当于间接提交了其他term的log entry

# 课堂记录
## 脑裂
之前提到的容错系统mapreduce、GFS、Vmware FT，都是单主节点。单主节点本身就是一个单点障碍，一旦出错就比较麻烦。
主节点出错的一个方式是split brain(脑裂)，也就是说系统中存在多个主节点，可能会导致系统中出现各种矛盾和错误
解决方案包括两种
1. 构建完美的网络：高成本
2. 人工解决问题：慢

## Raft
在系统中raft相当于底层服务，由上层应用调用。上层应用可以是多样的，例如kv数据库等。
client向raft的leader发送请求后，leader将该请求变成日志发送给集群中的其他server。
当**半数以上**的server拷贝了该日志后，raft向上层应用发送信息并commit该log entry。
commit信息会随着AppendEntries（心跳or下一条日志）发送给follower，follower也会执行该操作。
### 日志
在Raft中日志有以下作用：
1. **确定操作顺序**：leader收到不同操作后通过日志确定顺序
2. **缓存操作**：follower收到操作后缓存，等待leader发送commit信号；leader缓存操作，给follower重发信号
3. **状态恢复**：
