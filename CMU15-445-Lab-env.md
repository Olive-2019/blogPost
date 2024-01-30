---
title: CMU15-445-Lab 环境配置
date: 2024-01-30 16:07:30
tags:
- DB
- Ubuntu
- docker
- clion
---
在Ubuntu18.04上配置CMU15-445-Lab环境。由于实验环境需要Ubuntu22.04，因此需要使用docker来搭建实验环境。
<!-- more -->
# ssh登录
使用ssh登录到Ubuntu18.04
```bash
ssh -i dwf_dev_new root@172.21.11.36
```
# docker镜像

## dockerFile
使用实验自带的dockerFile来构造镜像，添加必要的依赖，如gdb等
```dockerfile
FROM ubuntu:22.04
CMD bash

# Install Ubuntu packages.
# Please add packages in alphabetical order.
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get -y update && \
    apt-get -y install \
      build-essential \
      clang-12 \
      clang-format-12 \
      clang-tidy-12 \
      cmake \
      doxygen \
      git \
      g++-12 \
      pkg-config \
      zlib1g-dev\
      ssh \
      openssh-server \
      vim\
      gdb

RUN echo "root:root" | chpasswd #修改root密码
RUN mkdir /var/run/sshd

CMD ["/usr/sbin/sshd", "-D"] 

```

## 构建镜像 启动容器

```shell
docker build . -t bustub ### 构建镜像
docker run -d --cap-add sys_ptrace -p127.0.0.1:2222:22 --name bustub bustub ###启动容器
docker exec -it bustub /bin/bash ### 进入容器
```

# 配置clion
连接到远程服务器，并配置编译环境
settings -> Build, Execution, Deployment -> Toolchains -> docker
# 测试
编译运行test/primer/中随意一个test文件
```bash
TEST(StarterTrieTest, Ctest) {
  std::cout << "fucking bustub";
}
```

