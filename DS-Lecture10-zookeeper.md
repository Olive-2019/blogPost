---
title: DS-Lecture10:zookeeper
date: 2023-11-04 10:35:46
tags:
---
zookeeper
<!-- more -->
# 论文
## 分布式协同
需要的内容：配置信息、不同节点的关系（如选举）、锁机制
实现方式：
1. 实现特定原语满足不同需求
2. 提供高级原语，可在高级原语基础上再构造低级原语

zookeeper走的第二条路线，提供一些API允许用户构造自己的原语，但是zookeeper不提供阻塞原语（锁可能导致差进程拖慢好进程）。
为了实现系统同步，zookeeper提供了wait-free的数据对象，wait-free数据对象是分层结构的。同时需要保证客户端访问FIFO和操作的linearizability。
## zookeeper 数据模型
zookeeper提供的数据对象称为znode。znode仿照inode，提供**层次化**的访问方式。与inode不同
1. znode存储在内存中
2. znode非叶结点也可以存储数据
3. znode存储的是元数据
### 节点类型

* 普通节点：客户端显式创建删除的节点，会被持久化
* 临时节点：仅在会话周期内存在，不会被持久化

顺序标志：允许客户端设置的，子节点的顺序标志不小于父节点

### watch机制
功能：当数据发生改变时，主动向客户端推送更改消息

特性：
* 一次性：仅在会话中有效，且仅推送一次

### 会话
客户端和zk连接后启动会话。客户端显式关闭或zk认定client存在故障时会话结束。
1. 超时时间。若超过该时间没有收到client的会话消息，则认定该client存在故障。
2. 会话内，客户端观察到的系统状态与其提交顺序完全一致
3. zk节点发生故障时，客户端会被无缝对接到另一个节点上（更换过程对客户端透明）

## zookeeper API
create(path, data, flags)：创建一个路径为 path 的 znode，并将数据 data[] 保存其中，然后返回新建的 znode 的名称。flags 用于指定 znode 的类型：常规节点，临时节点，以及设置顺序标记。
delete(path, version)：如果指定路径 path 下的 znode 的版本号和 version 匹配，则删除该节点。
exists(path, watch)：如果指定路径 path 下的 znode 存在则返回 true，否则返回 false。如果 watch 为 true，则当 znode 的数据发生变化时，客户端会收到通知。
getData(path, watch)：获取指定路径 path 下的 znode 的数据和元数据，例如版本信息。watch 的功能和 exists() 中的 watch 的功能一致，只不过如果当前节点不存在，则 watch 不会生效。
setData(path, data, version)：如果指定路径 path 下的 znode 的版本号和 version 匹配，则将数据 data[] 写入到该节点。
getChildren(path, watch)：返回指定路径 path 下的 znode 的子节点的名称。
sync(path)：阻塞等待直到在操作开始时所有进行中的更新操作都同步到了当前客户端所连接的服务端。path 参数当前未使用。
### 特点
* 异步：znode访问支持异步，保证异步回调函数的执行顺序与提交异步请求的顺序一致
* 无状态访问：不使用handle对znode进行访问，每次传输所有需要的数据
* 版本校验：允许客户端传入版本号，若不一致则拒绝请求

### 保证
* **线性化写**：写操作是原子性的、立即可见的，当写操作完成后，整个系统都可见
* **客户端FIFO请求顺序**：如果一个客户端按照顺序发送了多个请求，Zookeeper将按照相同的顺序将这些请求交付给服务端处理，而不会出现乱序。这个保证确保了客户端请求的有序性，使得客户端能够依赖于请求的顺序性进行正确的操作。

案例：leader-follower集群中，leader更改配置需要保证
1. 更改配置信息的过程中，配置信息不会被读取
2. leader在修改配置信息的过程中挂掉，改了一半的配置信息也不会被读取

方法：配置信息可用时，存在readyNode，leader修改配置信息时删除readyNode，并在修改后建立readyNode。虽然多个修改操作是异步发布的，但由于FIFO的保证，ready节点重新建立之前，修改操作必然完成。

另外两个小的保证：
* 可用性：半数以上服务器可用，集群可用
* 持久性：只要成功写，该状态改变就会被持久化

### 实现原语例子
1. 配置管理：简单的应用znode及其watch机制，保证可以获取最新配置
2. 汇合：用一个znode作为标志，多个worker进程建立时该znode建立，并监控该znode，该znode更新/被删除时，worker进程将杀死自己
3. 成员关系：在一个znode下建立子节点，每个member将自己的信息存储在该子节点中，当member挂掉时，该子节点被删除。想要全局信息时，打印所有子节点信息即可
4. 简单锁：创建znode即获得锁，其他进程监控该znode，会导致惊群效应
5. 无惊群效应的简单锁：创建znode时加上顺序号，最小者获取锁，其他进程watch比自己小1的znode
6. 读写锁：写锁即无惊群效应的简单锁，可共享读锁仅检测znode顺序号比自己小的写锁
7. 双重屏障：其实就是根据子节点数量进行同步

## zookeeper 实现原理
三个组件，请求处理器(request processor)， 原子广播(atomic broadcast)，复制数据库(replicated database)。
1. 请求到来时，zkServer将其放入request processor，request processor判断是否需要广播。
2. 若不需要广播（读请求），则从本地数据库获取数据返回
3. 若需要广播（写请求），则使用atomic broadcast将变更广播到集群中的所有服务器上
4. 收到半数follower的ack，则commit事务

每个zkServer都可以服务客户端，但是写请求会通过zab转发到leader上

### request processor
为了保证重复包也可以正常处理，leader将事务转化为**幂等**事务。
例如，将setData转换为setDataTXN，后者将包括修改后的数据，时间戳，修改版本号等信息，保证幂等性

### atomic broadcast(Zab)
* 请求流水线化，对次序的要求（由TCP实现）比常规原子广播更强。

* 2PC流程
1. 发送
2. 等半数以上服务器ack
3. commit

### replicated database
定期快照，从上次快照后开始replay日志。
zk采用模糊快照，生成快照时不锁定zk树，所以生成快照过程中也可能被修改。但是由于事务幂等，只要按序replay就可以达到原有状态



## zookeeper和client的交互
* 线性化写请求
* 本地化：只有与client相连的zkServer响应client的watch，同时client的读请求是本地zkServer响应的，故速率较高。读请求会有一个zxid，标记该服务器能看到的最后一个事务。
* 可用性：读操作可能返回过时的值，可以加上sync，根据FIFO，可以保证读到最新的值
* 持久性：如果zkServer的数据赶不上client的zxid，则不会建立会话，而是给该client找更新的zkServer
* 超时：zkServer一段时间内都没有收到client的请求即认为该client挂掉，故低活跃时期的client仍要给zkServer发心跳信息。client如果没有收到zkServer的回复，超时则切换到新服务器上。
