---
title: 分布式系统 容错VMware FT
date: 2023-10-03 00:26:51
tags:
    - mit 6.824
    - 容错
---
VMware 的容错系统，走指令复制的路子
<!-- more -->
# 论文阅读
本质上就是VMware的老本行，用VMM消除指令执行的不确定性，并通过传输指令的方式对系统进行复制
## 主备份容错
主备份服务器保持一致有两种方式：
1. **状态转移**：将primary的所有状态发送给backup，缺点是传输数据量大。
2. **备份状态机**：将primary的所有操作给backup一份，让backup称为primary的状态，传输数据量小。

## VMware
两个虚拟机分别是primary和backup，部署在不同机器上，共享磁盘。
虚拟机和物理机之间有一层虚拟机管理程序（VMM）hypervisor，VMM对硬件的模拟和控制可以将不确定的操作确定化，保证后续的replay。
primary和backup之间通过**日志**来实现备份状态机同步。VMM将primary的所有操作确定化为一条x86 CPU指令，并写出日志，通过logging channel传输到backup的log cache上。
## logging buffer和channel
primary和backup的状态备份通过**primary写出日志，backup读取日志**的方式进行。这个地方的写出、读取并不涉及磁盘，而是在内存中进行的，即logging buffer。
两个logging buffer之间的通信通过logging channel进行。
需要保证两者的logging buffer处于平衡状态，如果backup 的replay太慢，1.primary的logging buffer会爆，2.积存太多日志，primary失效后，需要等待backup较长时间执行replay操作。从外部看，系统响应时间较长。
## FT Protocol
logging channel的协议
### 基本要求
当primary失效时，backup之后向client发送的所有output要与primary一致
### Output Rule
为了避免primary在output中断的时候故障，backup没有接收到primary的状态
1. primary 在进行output之前，都把output相关信息发一份给backup，收到backup的ack以后才真正执行output操作
2. primary等待backup的ack期间也执行后续操作，只是推迟output而已

注：该协议无法保证重复输出，依赖于其他设施保证（如TCP检测重复数据包）
## 故障监测和响应
### 故障检测
primary和backup之间有三种信息可以用于进行故障检测，如果心跳和日志都超时了，就说明发生故障了
1. primary和backup之间的心跳信息（UDP）
2. logging channel的流量监测
3. backup给primary的ack
### 故障响应
backup检测到故障（长时间没有收到响应）后，并不能直接顶上primary的位置。因为直接顶上有脑裂的风险，同时存在两个primary对磁盘数据进行读写会造成数据损坏。
所以两个虚拟机需要通过共享磁盘获取对方是否故障的信息。具体做法是通过test-and-set原子操作竞争磁盘，拿不到磁盘锁的就不是primary。
若primary失效，那么backup顶上，并通过VMotion创建一个新backup，若backup失效，则创建一个新backup
## VM-FT和GFS的区别
本质上就是备份级别的问题，VM-FT是机器级的复制（low level 机器指令），GFS是应用级的复制（high level 数据库命令等）
1. VM-FT备份**指令**，可以为**所有服务**提供容错服务，且**一致性严谨**
2. GFS备份**数据**，**只针对数据存储**服务，备份策略更**高效**

# 课程记录
复制是容错的基本操作。但是复制并不能解决所有问题，主要解决的是fail-stop的错误，对软硬件的bug无能为力。
## 复制的两种策略
根据备份的内容和级别不同，可以分为以下两种备份策略
1. state transfer状态转移：将内存、磁盘等内容拷贝给backup，简单粗暴
2. replicated state machine复制状态机：将事件（指令）传递给backup，复杂

replicated state machine只是同步时成本低，但是当backup/primary失效时，仍是需要通过state transfer来创建新副本

## VMware FT工作原理
在两台物理机上部署VMM，VMM向两个异地虚拟机分配相同的配置（内存等），同一个LAN里还有一些客户端和共享磁盘（共享磁盘和客户端可以大致认定相同）
由primary的VMM将外部输入转发给backup，backup的VMM知道自己是backup，所以会对backup虚拟机的输出进行丢包
* 当primary发生故障时，backup的VMM会在一些防止split-brain的操作后充当primary，不再丢包，执行primary的转发功能，并控制生成backup
* 当backup发生故障时，primary的VMM会控制生成新的backup，并转发数据包到新的backup上
## 不确定事件
不确定事件可以有以下几类
1. 客户端输入：不可控
2. 怪异指令：随机数、获取当前时间、获取计算机id
3. 多CPU并发的指令执行顺序

这些事件都要通过log channel发送到backup，老师猜测的日志内容应该包括
1. 事件的指令序号
2. 日志类型：数据包or怪异指令？
3. 数据：数据包就是普通数据，怪异指令就是提供primary执行该指令后的系统状态，方便backup伪造该指令
