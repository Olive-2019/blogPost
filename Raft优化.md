---
title: Raft优化
date: 2023-11-08 12:27:45
tags:
---
raft协议的优化方向
<!-- more -->
# 读优化
读操作有两种方式，从leader读/从任意节点读。前者在spilt brain时存在一致性问题，后者任何时候都存在一致性问题（未同步）。故只讨论从leader上读的优化。

## read index
read index所解决的核心问题有两个：
1. leader是否有效
2. 前面的logEntry是否提交

具体方式为：
1. 将当前未commit的logEntry提交确保系统状态已更新
2. 提交后发起一轮心跳确保当前leader是唯一有效的
3. 回应client的读操作


对比原始raft算法优点：
1. 不必将读操作变成logEntry复制到集群中的所有节点上和等待commit，节约了至少一次commit的时间

## lease read
核心是leader**计算最小独立掌权时间**。leader维护一个租约机制，当租约未过期时，可以在commit了所有entries以后直接返回，不必发心跳。
租约机制通过选举超时时限和发布心跳时间计算最小独立掌权时间，即$leaseTime = start+timeout$。
由于不同CPU的时钟会有误差，故使用$leaseTime = start+\frac{timeout}{clock\ drift\ bound}$，但是实际上这个算法仍然存在问题，故不会被作为默认算法。

对比原始read index优点：
不用心跳

# 选举优化
raft选举过程中仍存在多个问题
## prevote
解决的核心问题：
出现网络分区时，小分区里的节点无法选举成功会不断尝试，并增加term。当网络分区恢复时，小分区节点和leader的通讯会导致leader卸任。但是leader不应该卸任。

解决方案：
在真正的选举之前进行一轮预选举，如果能成功再正式发起选举，否则放弃本轮选举，等待下一个election timeout
## priority election
解决的核心问题：
raft不支持优先级选举

解决方案:
引入节点优先级和全局优先级，选举前比一比

具体方案：
1. 每次心跳都更新全局最高优先级
2. 超时时比较节点优先级和全局最高优先级，若节点优先级>=全局最高优先级，则开启选举
3. 若选举超时仍未选出leader，则$全局最高优先级 = max(1, 全局最高优先级 * 0.8)$

存在的问题：
当最高优先级不存在时，选举时可能会浪费一轮选举时间

# Leader的自我管理
传统的raft几乎不存在leader的自我管理，只在接收到更新的term时退出当前任期

## Leader变更

1. 停止apply
2. 给新leader发timeoutNow
3. 新leader进入candidate状态，开始竞选
## Leader检测
心跳加上超时标签，检测断联节点。断联节点达到大多数时，该状态机即认为已经废了，不再接受输入。

