---
title: DS_Lab2
date: 2023-10-13 19:28:07
tags:
    - 分布式
    - Raft
---
mit6.824 lab2。lab2 是后续实验的基础，实现一个Raft，lab3实现的KV是在Raft上的。

<!-- more -->

# 论文中给出的算法

## State状态

### 需要持久化的state（所有server）

1. **currentTerm**：server所处的term，也是该server所得知的最新term
2. **votedFor**：如果是null，则该term内尚未投票，如果是candidateID，则该term中的选举已经投票给了该candidate（不能再投票）。
3. **log**[]：当前server的所有log entries，每一条log entry包含命令和term编号

### 易失state（所有server）

1. **commitIndex**：最大的已经commit的log entries index
2. **lastApplied**：最新加入的log entries index

### 易失state（leader独有）

1. **nextIndex**[]：需要发给每一个follower的下一条log entry（初始化值是leader的最后一个log entry的下一个值）
2. **matchIndex**[]：每个follower当前匹配到哪一条log entry（初始化值为0）

## AppendEntries RPC

leader->follower

### 参数

1. **term**：leader的当前term编号
2. **leaderId**：leader的id，方便follower将收到的信息转发给leader
3. **prevLogIndex**：前一个log的index，方便follower确认一致性
4. **prevLogTerm**：前一个log的term，功能同上
5. **entries**[]：需要拷贝的多条log entry，心跳信息会是空
6. **leaderCommit**：leader提交的日志 entry index

或许可以带上leader自己的log entries，便于找nextIndex

### 返回值

1. **term**：follower所处的term，便于leader确认自己的地位
2. **success**：用于标志follower是否成功复制log entries。如果prevLogIndex/Term不匹配就会返回false

### Follower的处理逻辑

1. 直接返回false：term< currentTerm or prevLogIndex/Term对应的log不存在
2. 如果存在冲突entry，即相同的index，不同的term（就是leader没有的数据），删掉现有的entries并写入leader给的数据
3. 将不存在的entries追加到系统中
4. 当leaderCommit > commitIndex时，更新commitIndex = min(leaderCommit, 目前最新entry的index)

## RequestVote RPC

candidate->follower

### 参数

1. **term**：当前竞选leader的candidate的term
2. **candidateId**：当前竞选leader的candidate的ID
3. **lastLogIndex**：最后一条log entry的index
4. **lastLogTerm**：最后一条log entry的term

### 返回值

1. **term**：follower的term，如果candidate没有这个大，将会退回到follower状态
2. **voteGranted**：该follower是否投票给该节点

### follower处理逻辑

1. 直接返回false：term < currentTerm
2. 如果 （votedFor == null || votedFor == candidateId） && candidate的log比当前节点新，投票给该节点，否则拒绝该节点

## Server Rules

所有server都需要执行的：

1. if commitIndex > lastApplied: commitIndex之前的日志将会被执行，更新lastApplied到commitIndex
2. if RPC请求或回复包含term，且term > currentTerm，则更新currentTerm，并转换到follower

### Follower

1. 回应candidate和leader的RPC调用（自己注册一个函数）
2. 选举计时器超时参加选举

### Candidate

1. 开始参加选举的动作：currentTerm自增、给自己投票、随机一个新的选举计时器、给其他server发送RequestVote RPC
2. if 收到过半服务器的投票：成为leader
3. if 收到新leader的AppendEntries RPC：成为follower
4. if 选举计时器超时：重新参加选举

### Leader

1. 当选后：立即给其他server发送心跳信息（空AppendEntries），防止超时重新选举
2. 从client处接收到command：向本地log写入entry，当该log entry commit时，向client回复信息
3. if last log entries index >= 某一个follower的nextIndex，向该follower发送AppendEntries RPC新增其日志。如果成功，则更新nextIndex和matchIndex，如果失败，则降低其nextIndex重试
4. if 存在N>commitIndex && N <= 过半数的 matchIndex && index为N的日志的term==currentTerm， then 更新commitIndex = N

# 实验

lab2 是后续实验的基础，实现一个Raft，lab3实现的KV是在Raft上的。

## Lab2A

实现一个Raft选举算法

### 类架构
```mermaid
classDiagram
    FileOperator <|-- ServerAddressReader
    State <-- AppendEntries
    State <-- RequestVote
    State <-- TimeOutCounter
    State <-- LogEntry
    State <-- ServerAddressReader
    Raft <-- State
    State <|-- Leader
    State <|-- Follower
    State <|-- Candidate
    POJO <|-- AppendEntries
    POJO <|-- RequestVote
    POJO <|-- LogEntry
```

### 线程梳理

Raft运行主线程，不断地切换状态，切换的过程是由上一个状态创建新状态并返回其指针
状态的run函数在Raft主线程中运行，会创建多个线程。以下按状态机类型梳理线程

#### 所有server，尤其是Follower

1. 计算超时的线程timeOut。（单独开一个类来承担这个职责）Sleep超时计时器的时间后检测是否有收到过AppendEntries RPC（这里需要一个标志位）。如果收到过，则复位标志位，并Sleep，如果没有收到过，则终止当前状态（该状态会创建candidate状态返回），退出该线程。这里的终止状态需要考虑到RPC Server的退出，根据[rest_rpc在GitHub的issue](https://github.com/qicosmos/rest_rpc/issues/51)中提到的，可以通过智能指针创建server，退出时将其置为nullptr。
2. 等待接收AppendEntries。开一条线程用于RPC通信，接收其他server（此处指leader）的AppendEntries请求，这里需要对log相关变量加锁，避免产生幻觉
3. 投票线程RequestVote。开一条线程用于RPC通信，接收其他server（此处指candidate）的RequestVote请求，这里需要对vote相关变量加锁，避免产生幻觉
```mermaid
stateDiagram-v2
    [*] --> timeoutCounter
    timeoutCounter --> [*]
    [*] --> receiveAppendEntries
    receiveAppendEntries --> [*]
    [*] --> receiveRequestVote
    receiveRequestVote --> [*]
```
#### Leader

1. 等待client的start。开一条线程用于RPC通信，接收client的command，这里需要对log相关变量加锁，避免产生幻觉
2. 发送AppendEntries的线程。调用其他server的AppendEntries，并异步等待结果。
```mermaid
stateDiagram-v2
    [*] --> timeoutCounter
    timeoutCounter --> [*]
    [*] --> receiveAppendEntries
    receiveAppendEntries --> [*]
    
    [*] --> receiveRequestVote
    receiveRequestVote --> [*]

    [*] --> receiveStart
    receiveStart --> [*]

    [*] --> sendAppendEntries
    sendAppendEntries --> [*]
    
```
#### Candidate

1. 发送RequestVote的线程。调用其他server的RequestVote，并异步等待结果。
```mermaid
stateDiagram-v2
    [*] --> timeoutCounter
    timeoutCounter --> [*]
    [*] --> receiveAppendEntries
    receiveAppendEntries --> [*]
    
    [*] --> receiveRequestVote
    receiveRequestVote --> [*]

    [*] --> sendRequestVote
    sendRequestVote --> [*]
    
```