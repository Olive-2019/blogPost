---
layout: ds_lab2
title: 分布式系统 GFS
date: 2023-09-30 14:51:56
tags:
    - mit 6.824
    - 分布式系统
    - GFS
---
Google File System，有点古老的技术，但技术仍存在参考意义
<!-- more -->

# 论文阅读
## 背景
Google有大量数据存储的要求，所以诞生了GFS这个大规模分布式文件存储系统。根据工作负载和技术环境，GFS和传统文件系统有以下区别：
1. **组件故障是常态** 分布式设备廉价容易出故障，而且不易恢复。
2. **文件大**
3. **修改==追加** 不存在对文件的随机写，一旦写入就不再修改，通过追加的方式进行修改
4. 多客户端可能同时对文件进行原子追加操作
5. 高带宽比低延迟更重要
## 架构
GFS集群由1个**Master**、n个**chunk server**、m个client组成。
这些东西都跑在普通的Linux服务器上。chunk server和client是可以在同一台服务器上跑的。
### 文件
1. 文件会被分块，每一块叫做一个chunk。
2. 每个chunk有一个64位的唯一标识chunk handle，这个标识是在分块的时候由master产生的。
3. 文件存储在chunk server上，通过chunk handle和字节范围读写。
4. 文件会存在副本，可以设置不同命名空间中文件的不同副本数量
### 存储与交互
master存储文件系统的元信息，包括以下内容：
1. 名字空间 namespace（文件名）
2. 文件->块 映射信息
3. chunk位置信息（不需要持久化）
4. 访问控制信息 
master还控制了系统活动，如
1. 租约管理
2. 孤儿块垃圾回收
3. chunk server之间的块迁移
4. master用心跳信息管理chunk server状态
client绝不跟master要数据，而是向master要元信息，发送一个包含文件名和块索引的请求，master返回chunk handle和副本位置；然后client再向chunk master请求具体文件。
这么做保证了单master的可行性
# 课程记录
## 分布式存储的难点
追求**高性能**->将文件**分片**放在多个服务器上->更容易发生**故障**->需要**容错**措施->建立**副本**->追求**一致性***->**低性能**
## GFS特点
1. **单数据中心**：只在一个数据中心运行
2. **工业软件**：不面向普通用户
3. **大规模顺序读写**：不支持随机访问，更关注吞吐量（带宽）而不是时延
4. **弱一致性**：保证性能
5. **单master**
## GFS结构
master 存储元信息，chunk server 存储数据信息。两种服务器相互隔离
### Master存储信息
master中主要有两张表：
1. filename-> chunk handle（需要**持久化**）
2. chunk handle->chunk servers
 * chunk servers服务器列表：每个Chunk存储在哪些服务器上
 * chunk版本号：每个Chunk当前的版本号（需要**持久化**）
 * 主chunk地址：所有对于Chunk的写操作都必须在主Chunk（Primary Chunk）上顺序处理，主Chunk是Chunk的多个副本之一。所以，Master节点必须记住哪个Chunk服务器持有主Chunk。
 * 主chunk租约到期时间：主Chunk只能在特定的租约时间内担任主Chunk，所以，Master节点要记住主Chunk的租约过期时间。

 持久化的方式是**写日志**和**打检查点**。
 * 加入新chunk或者修改chunk版本时需要向磁盘写一条日志
 * 定时打checkpoints，将状态转移到磁盘中，恢复时选择最近的checkpoints，从该checkpoint后的日志开始恢复，而不必从猴年马月开始恢复。

## Read读数据
### client和master 元信息交互
1. client 发送[filename文件名, offset偏移量]给master
2. master从filename->chunk handle中查询**chunk handle**，offset / chunk size 就是chunk handle在数组中的index，从而可以获取chunk handle
3. master从chunk handle->chunk servers中查询**chunk server**的信息
4. master将[chunk handle, chunk server]发送给client
client 会缓存 master返回的信息，因为可能会持续访问这个chunk
如果跨chunk访问，client端会自动分成两个读请求发给master
### client和chunk server 数据交互
1. client 将[chunk handle, byte range]发送给选中的chunk server
2. chunk server根据chunk handle找到文件，根据byte range获取数据，发回给client

## Write写数据
GFS只追加数据，而不随机写数据。写数据时需要向持有primary chunk（主副本）的chunk server写入，当不清楚primary chunk时，需要进行以下操作
### 确定primary
1. 找到**最新副本**：chunk版本号==master中记录chunk的版本号
2. 选中**primary chunk**：选择其中一个chunk server作为primary chunk，其他的就是secondary。primary是有租约的，超时过期就失去了primary的地位，为了避免同时出现两个primary，就算master联系不上primary，也会等待租约到期。
3. 版本号自增
4. 告诉chunk server，primary、secondary 版本号
5. master写入版本号

注意：
 * 版本号是master指定primary时才会产生的
## 客户端和primary的交互
1. 客户端将数据往chunk server（primary & secondary）发
2. chunk server会将数据写入临时区，并给client返回确认信息
3. client给primary发送确定信息，意义为：你和所有secondary都有了数据，请写入chunk末尾
4. primary调度并发请求，并以某种顺序，保证一次只执行一个请求，给所有secondary发送写请求
5. 所有chunk server收到写请求后，将临时区的数据往chunk里写，完成后给primary发送确认信息，意义为：已经成功写入数据
6. 如果primary收到了所有secondary的确认消息，则返回给client确认消息，否则，若存在一个异常（secondary没有回复or回复错误信息），primary向client报告失败
7. client收到失败信息后会重新发起数据追加操作

写文件失败的一些事项
 * 部分chunk server确实写入了，所以存在读到新数据的可能
## GFS一致性
如果整套流程成功走下来，则系统各副本一致。
假如存在个别副本失败，则重新发一遍请求，在新的位置写入，所以读不同副本可能会读到不同的数据（有的可能有多套数据，有的可能只有一套，假如client也挂了，那可能存在一套数据都没有的副本）
多副本不一致，设计方式简单，但是容易出错。（有点拉）