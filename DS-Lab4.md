---
title: DS_Lab4
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

# 详细设计
ShardCtrler需要实现以下功能
* Join : 新加入的Group信息（PUT、Append）
* Leave : 哪些Group要离开（Delete）
* Move : 将Shard分配给GID的Group,无论它原来在哪（Delte Put）
* Query : 查询最新的Config信息（Get）
