---
title: rest_rpc
date: 2023-10-23 14:33:50
tags:
    - 通信
    - C++
---
C++开源rpc库的介绍和使用方法
<!-- more -->
# RPC
RPC（Remote Procedure Call Protocol）远程过程调用协议。一个通俗的描述是：客户端在不知道调用细节的情况下，调用存在于远程计算机上的某个对象，就像调用本地应用程序中的对象一样。比较正式的描述是：一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。
# 配置过程
rest_rpc支持C++调用，其配置过程如下
## 安装boost
1. 从[boost官网](https://www.boost.org/)上下载最新boost的zip版本
2. 将压缩包解压到合适的路径
3. 双击根目录下的bootstrap.bat文件，生成b2.exe
4. 双击b2.exe运行
5. VS studio 配置项目属性->VC++目录
(1) "包含目录": boost的根目录，例:C:\boost_1_79_0
(2) "库目录": stage下的链接库目录，例:C:\boost_1_79_0\stage\lib
6. 配置属性->链接器->常规:"附加库目录":同上面的"库目录"，例:D:\my_workspace\C_program\C_boost\boost_1_79_0\stage\lib
## 下载rest_rpc
```bash
git clone https://github.com/qicosmos/rest_rpc
```
# 使用方法
## 基础
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
## 优雅的退出方法
使用智能指针指向server，将其reset为nullptr时自动析构server，即可退出run
创建过程
```C++
unique_ptr<rpc_server> server;
server.reset(new rpc_server(startAddress.second, 6));
server->register_handler("func_greet", hello);
server->run();//启动服务端，此处会阻塞线程
```
释放过程
```C++
server->reset(nullptr);
```

## 调用类成员函数的方法
调用register_handler时需要按这个格式来，func是函数名，每个参数都需要写出来
```C++
server->register_handler("func", [this](rpc_conn conn, string arg) {
		this->func(std::move(arg));
	});
```
## 异步调用，等待返回信息
第一个参数是函数指针，后面的参数是参数列表，类成员函数记得第一个是对象指针
```C++
shared_future<type> returnVal = async(funcPtr, args...);
```
