---
layout: 
title: 分布式系统 MapReduce
date: 2023-09-17 23:12:24
tags: 
 - MapReduce
 - 分布式系统
 - mit 6.824
---
第一课，分布式系统概述与mapreduce相关内容
<!-- more -->
# 论文阅读
## MapReduce 背景
MapReduce简介：MapReduce是一种编程模型，用于处理和生成大规模数据集。用户指定一个map函数，用于处理输入的键/值对，生成一组中间的键/值对；以及一个reduce函数，用于合并具有相同中间键的所有值。
MapReduce实现：MapReduce利用用户定义的map和reduce函数，自动地将计算并行化并在大规模的集群上执行。运行时系统负责分割输入数据，调度程序在多台机器上执行，处理机器故障，以及管理机器间的通信。
MapReduce应用：MapReduce可以表达许多实际中的任务，例如分布式grep、URL访问频率统计、反向链接图构建、词向量生成、倒排索引构建和分布式排序等。
MapReduce类型：MapReduce的输入和输出都是键/值对的集合，但它们可以来自不同的域。中间的键/值对也是来自输出域的。用户可以使用字符串来转换不同类型的数据。
MapReduce扩展：MapReduce提供了一些扩展功能，例如自定义分区函数、排序保证、组合函数、输入和输出类型、副作用处理、错误记录跳过、本地执行、状态信息和计数器等。
MapReduce与其他并行计算模型的比较：MapReduce是一种简化和提炼了一些并行计算模型的编程模型，它根据Google的大规模实际计算的经验，自动地实现了并行化、容错、局部性优化和负载均衡。
MapReduce与其他分布式系统的比较：MapReduce与一些提供高级抽象的分布式系统不同，它利用了一种受限的编程模型，使得用户程序可以自动地被并行化和容错。MapReduce还与一些针对特定应用的分布式系统不同，它提供了一个通用的库，可以用于各种类型的数据处理任务。
MapReduce的灵感来源：MapReduce借鉴了一些技术，如主动磁盘、网络排序、分布式队列、数据分散和重复执行等，来提高性能、可扩展性和容错性。
MapReduce的结论：MapReduce编程模型在Google被成功地用于多种目的，因为它易于使用、适用于多种问题、能够扩展到大量机器，并且有效地利用了机器资源。1作者从这项工作中学到了一些经验，如限制编程模型可以简化并行化和容错、网络带宽是稀缺资源、重复执行可以减少非均匀性的影响等。
## MapReduce 运行逻辑
可以理解为，将原有的多节点并行计算相关的操作封装了，只暴露了两个函数map和reduce，这两个函数由用户编写，注册到MapReduce框架中。
### 基本思想
split（数据分割）->map（数据转换）->shuffle（数据分类）->reduce（合并数据）
MapReduce把所有的计算都拆分成两个基本的计算操作，即Map和Reduce。其中Map函数以一系列键值对作为输入，然后输出一个中间文件。这个中间态是另一种形式的键值对。最后，Reduce函数将这个中间态作为输入，计算得出结果。
### 运作规则
1. MapReduce客户端会将输入的文件会分为M个片段，每个片段的大小通常在 16~64 MB 之间。然后在多个机器上开始运行MapReduce程序。
2. 系统中会有一个机器被选为Master节点，整个 MapReduce 计算包含M个Map 任务和R个 Reduce 任务。Master节点会为空闲的 Worker节点分配Map任务和 Reduce 任务
3. 执行Map任务的 Worker开始读入自己对应的片段并将读入的数据解析为输入键值对。然后调用由用户定义的 Map任务。最后，Worker会将Map任务输出的结果存在内存中。
4. 在执行Map的同时，Map Worker根据Partition 函数将产生的中间结果分为R个部分，然后定期将内存中的中间文件存入到自己的本地磁盘中。任务完成时，Mapper 便会将中间文件在其本地磁盘上的存放位置报告给 Master。
5. Master会将中间文件存放位置通知给Reduce Work。Reduce Worker接收到这些信息后便会通过RPC读取中间文件。在读取完毕后，Reduce Worker会对读取到的数据进行排序，保证拥有相同键的键值对能够连续分布。
6. 最后，Reduce Worker会为每个键收集与其关联的值的集合，并调用用户定义的Reduce 函数。Reduce 函数的结果会被放入到对应的结果文件。
7. 当所有Map和Reduce都结束后，程序会换新客户端并返回结果。
## 分布式容错方案
分布式系统不可避免地要考虑容错的问题，在MapReduce中，容错也考虑Master和Work两种情况。
### Master出错
Master节点会定期地将当前运行状态存为快照，当Master节点崩溃，就从最近的快照恢复然后重新执行任务。
### worker出错
Master节点会定期地Ping每个Work节点，一旦发现Work节点不可达，针对其当前执行的是Map还是Reduce任务，会有不同的策略。
1. Map任务，无论任务已完成或是未完成，都会废除当前节点的任务。Master会将任务重新分配给其他节点，同时由于已经生成的中间文件不可访问，还会通知还未拿到中间文件的Reduce Worker去新的节点拿数据。
2. Reduce任务，由于结果文件存在GFS中，文件的可用性和一致性由GFS保证，所以Master仅将未完成的任务重新分配。
## MapReduce优化策略
简单来说就是调节负载
如果集群中有某个 Worker 花了特别长的时间来完成最后的几个 Map 或 Reduce 任务，整个 MapReduce 计算任务的耗时就会因此被拖长，这样的 Worker 也就成了落后者。MapReduce 在整个计算完成到一定程度时就会将剩余的任务即同时将其分配给其他空闲 Worker 来执行，并在其中一个 Worker 完成后将该任务视作已完成。

# 课堂笔记
## 分布式系统需要克服的问题
并行、容错性、通信物理距离、数据安全与隔离问题
分布式系统的并发和错误是关键难点
## 分布式系统的基础架构
分布式的基础架构包括存储、通信、计算三个模块；希望能有抽象的接口，让分布式接口更接近非分布式的接口
### 存储
希望更抽象，容错、高性能
### 通信
网络通信 可靠性 详细内容见MIT6.829
### 计算
类似mapreduce
## 分布式系统所需要的技术
1. RPC：掩盖在非可靠网络上的事实
2. 线程：并发
3. 锁：并发
## 分布式系统的性能目标
### 可扩展性
性能可扩展性，加硬件设备就能增加同规模的性能
### 容错性
1. 可用性：在出现了不可避免的故障后，系统仍能正常工作（冗余）
2. 可恢复性：出现故障后，系统可以通过各种方式恢复到正常状态（非易失存储）
### 一致性
一致性包括强一致性和弱一致性，各有优劣
1. 强一致性：每次读出的数据都是最新的
2. 弱一致性：允许读出的数据没有被更新
强一致性代价高，需要的通信成本高。尤其是副本数据存储物理距离较远时，通信成本将会限制强一致性的保证。所以弱一致性是工业界常用的技术。
## MapReduce
### 工作方式
将分布式问题分为map和reduce两个函数，map将输入转换成中间键值对，reduce将中间键值对进行聚合计算。
整个mapreduce的过程称为一个job（作业），一次mapreduce的过程称为一个task（任务）。
### map函数
```c++
/*
map参数解释：key通常是不需要的，例如文件名、行号等，value是需要处理的，如文件实际内容
*/
map(key, value) {
    for each word in value:
        emit(word, 1);
    //对于value里的每一个单词，都需要将其组成一个新的key-value对发送
    //map函数里的emit接受两个参数，就是key value
}
```
### reduce函数
```c++
/*
reduce的输入既是map函数的输出，相同的中间key会发送给同一个reduce函数
*/
reduce(key, value) {
    emit(len(value));
    // reduce里的emit通常只有一个输出参数
}
```

# 实验
在进行了一部分的go尝试后，虽然听说lab2用C++不好写，但还是决定拿起我的C++（果真是最爱），go语法有点怪怪的
## 环境搭建
在Windows物理机上使用wsl2安装Ubuntu18.04，按照[课程指引](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html) 配置环境，值得注意的是，clone下来的代码跑不动的时候需要修改包的相对路径
## 实验大体要求
1. 两个程序 Coordinator(协调者)和Worker(工作者)，系统中有一个Coordinator和多个Worker进程并行执行。
2. 进程间通信使用**RPC**进行，通信主要存在于coordinator和worker之间，通信内容是worker和coordinator之间的任务分发、数据来源地址、任务执行、输出写入地址
3. 当coordinator发现worker长时间没有完成任务（10s）时，需要将该任务发布给其他worker
## 详细规则
1. map阶段应该将中间键分到不同的组里，将这些组传给**nReduce**个reduce任务。nReduce应该是MakeCoordinator()的参数（目前理解：建立Coordinator的时候就得指定有多少个reduce任务）
2. worker执行reduce任务时输出文件名应该统一成mr-out-X（X是reduce任务的编号，[0, nReduce - 1]）
3. mr-out-X文件中每行包含一个Reduce函数产生的结果，应该使用Go中的“%v %v”格式，来写入reduce后的key和value
4. worker执行Map任务时，输出存到当前目录的文件，这样之后该文件作为reduce工作的输入能够被读到。
5. 当所有工作结束，worker进程也应该退出。一个简单的办法是，每次调用call()函数时查看其返回值，如果返回值显示不能正常与coordinator通信，那么就假定工作已经结束且coordinator已经退出，因此worker也可以结束。这取决于你的设计，也可以设计"please exit"作为coordinator给worker的伪任务，来提提示worker的退出。
## 论文细节描述
1. MapReduce库首先将**输入文件分成M个片段**，每个片段通常为16到64兆字节（MB）（用户可以通过可选参数进行控制）。
2. 在机群上启动多个程序。其中一个程序是特殊的，Master。其余的程序是由Master分配工作的Worker。有M个map任务和R个reduce任务需要分配。Master选择空闲的Worker，并为每个工作程序分配一个map任务或reduce任务。
3. 被分配map任务的Worker读取相应输入片段的内容。它从输入数据中**解析**出键/值对，并将每个键/值对**传递**给用户定义的map函数。Map函数生成的中间键/值对在内存中进行缓冲。
4. 定期将缓冲的键/值对按照分区函数**划分为R个区域**（需要排序吗？），并将这些缓冲区在本地磁盘上的位置传递回Master，由Master负责将这些位置转发给reducer。
5. 当reducer收到Master关于这些位置的通知时，它使用**远程过程**调用从map工作程序的本地磁盘读取缓冲数据。当reducer读取所有中间数据时，它按照中间键对其进行**排序**，以便将相同键的所有出现组合在一起。如果中间数据量太大无法放入内存，则使用外部排序。
6. reducer遍历排序后的中间数据，并对遇到的每个**唯一中间键**传递该键和相应的一组中间值给用户定义的reduce函数。Reduce函数的输出附加到此reduce分区的最终输出文件中。
## 实验梳理
### 结构
1. Coordinator项目里有Coordinator和WorkerState，不必关心mapper和reducer的具体任务，只需要做分配调度的工作。其中Coordinator依赖WorkerState完成调度任务，WorkerState管理单个Worker
2. worker项目里有Worker和Task及其子类，根据调用runTask的参数决定运行mapper还是reducer的任务
### 功能
1. Cooridinator不需要多实例运行。Cooridinator里需要创建一个worker池，将注册的不同worker放池子里，需要分配map、reduce任务时可以在池子里选一个worker运行，并修改其状态，该worker池需要考虑并发问题，写操作时加了一把大锁。
2. Worker项目需要多实例运行，得多开，运行时需要传入参数决定其端口。
3. Cooridinator和Worker通信时，注意端口不能硬编码，减少幻数
4. 任务**Task**可以作为一个父类，**Map**和**Reduce**作为子类，实现run接口，由于传入参数不一样，故需要解析参数的不同接口parseArgs。
## RPC补充信息
RPC（Remote Procedure Call Protocol）远程过程调用协议。一个通俗的描述是：客户端在不知道调用细节的情况下，调用存在于远程计算机上的某个对象，就像调用本地应用程序中的对象一样。比较正式的描述是：一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。
本项目使用rest_rpc，配置过程如下
### 安装boost
1. 从[boost官网](https://www.boost.org/)上下载最新boost的zip版本
2. 将压缩包解压到合适的路径
3. 双击根目录下的bootstrap.bat文件，生成b2.exe
4. 双击b2.exe运行
5. VS studio 配置项目属性->VC++目录
(1) "包含目录": boost的根目录，例:C:\boost_1_79_0
(2) "库目录": stage下的链接库目录，例:C:\boost_1_79_0\stage\lib
6. 配置属性->链接器->常规:"附加库目录":同上面的"库目录"，例:D:\my_workspace\C_program\C_boost\boost_1_79_0\stage\lib
### 下载rest_rpc
```bash
git clone https://github.com/qicosmos/rest_rpc
```
### 使用
1. 在server和client端都需要引入rest_rpc.hpp，此处需要注意其目录，如`#include "../rest_rpc/include/rest_rpc.hpp"`
2. 引入名字空间`using namespace rest_rpc;`
3. server端：
```c++
//第一个参数必须得是conn
string hello(rpc_conn conn, string name) {
	/*可以为 void 返回类型，代表调用后不给远程客户端返回消息*/
	return ("Hello " + name); /*返回给远程客户端的内容*/
}
int main() {
	rpc_server server(9000, 6);
	server.register_handler("func_greet", hello);
	server.run();//启动服务端

	return 0;
}
```
4. client端
```c++
try {
    /*建立连接*/
    rpc_client client("127.0.0.1", 9000);// IP 地址，端口号
    /*设定超时 5s（不填默认为 3s），connect 超时返回 false，成功返回 true*/
    bool has_connected = client.connect(5);
    /*没有建立连接则退出程序*/
    if (!has_connected) {
        cout << "connect timeout" << endl;
        exit(-1);
    }

    /*调用远程服务，返回欢迎信息*/
    string result = client.call<std::string>("func_greet", "Lam");// func_greet 为事先注册好的服务名，需要一个 name 参数，这里为 Hello Github 的缩写 HG
    cout << result << endl;

}
/*遇到连接错误、调用服务时参数不对等情况会抛出异常*/
catch (const exception& e) {
    cout << e.what() << endl;
}
```
