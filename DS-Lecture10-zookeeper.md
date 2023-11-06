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

## 