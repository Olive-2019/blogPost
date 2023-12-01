---
title: DS_Lab4 Shard
date: 2023-11-15 11:58:38
tags:
---
mit 6.824 lab4 容错分片KV数据库
<!-- more -->
# 需求概述
需要实现可分片的容错KV数据库，容错性和KV数据库由前两个lab实现，本lab实现的是**分片管理**。
此处KV数据库称为ShardKV，分片信息由ShardCtrler（分片管理器）管理。
# 概要设计

由一个容错的KV数据库集群维护**config**。config是分片shard-分片所在ShardKV集群ip的映射关系，如分片`1(1 mod 10)-127.0.0.1:7890`。
ShardKV和client都需要向ShardCtrler请求config信息，client需要知道自己当前使用的key被存在了哪个集群中，应该向哪个集群请求服务；而ShardKV需要根据config信息的变化，把自己集群的数据迁移到其他集群中。
迁移:
* 迁移过程由原有数据主发起，只有原有数据主才知道这个分片有多少数据。
* 迁移期间，两个集群都不对外提供服务，等待迁移结束才重新对外提供服务，保证强一致性

## ShardCtrler
ShardCtrler对于配置信息需要实现以下功能
* Join : 新加入的Group信息（PUT、Append）
* Leave : 哪些Group要离开（Delete）
* Move : 将Shard分配给GID的Group,无论它原来在哪（Delete Put）
* Query : 查询最新的Config信息（Get）

### 代码修改
KVServer增加Delete
## ShardKV
每隔一段时间向ShardCtrler要求config信息，与自己的对比，若相同则不用处理，若不同则需要暂时关闭服务进行处理
处理逻辑：
1. 若自己的shard数量减少了，则向ShardCtrler请求减少的shard新的所在位置，向该集群连续发送多个PUT请求和一个服务开启get请求
2. 若自己的shard数量减少了，则关闭get服务，等待PUT命令和开启服务请求

# 详细设计
将转移数据的影响降低到shard粒度上
## ShardCtrler
使用KV数据库存储config，使用双向map存储
### config 数据特征
group-shard 1-n
### config 存储形式
shardid从0开始计数，groupid从1开始计数
shardid --> groupid（从0开始递增）
-groupid --> groupAdress（即从-1开始递减）
## ShardKV
需要考虑在哪一层做这些处理
内存保存当前的所有shard，不持久化（没必要，每次去shardCtler拉就行）
轮询拉取本group的最新配置信息，若有变更，则进入服务迁移过程：(只检测是否有分片被转移即可)
### 存在新分片
不处理，让持有分片的group发送命令，待其传递完数据再将shardID加入本地配置中（由持有group发送该命令）
### 分片被转移
多个shard可以并发执行
1. 将该shardID从内存配置中删除（相当于对外关闭该shard的服务）
2. 向shardCtrler请求分片的新地址
3. 向新地址发送该shard的所有数据
4. 向新地址发送添加shardID的命令
5. 删除本group的数据
