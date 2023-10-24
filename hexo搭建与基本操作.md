---
title: hexo搭建与基本操作
date: 2023-10-20 15:11:53
tags:
    - hexo
---
hexo + GitHub搭建个人主页
<!-- more -->
[hexo文档主页](https://hexo.io/zh-cn/docs/index.html)
# 基本环境搭建
## nvm
### 下载
在[GitHub](https://github.com/coreybutler/nvm-windows/releases)上下载最新版本的nvm，[目前最新版链接](https://github.com/coreybutler/nvm-windows/releases/download/1.1.11/nvm-setup.zip)
### 安装
1. 解压，运行`nvm-setup.exe`，可以一路next，也可以手动指定安装路径
2. 安装完成后，命令行输入`nvm version`检查是否成功安装
## node
也可以单独安装node和npm，但是比较喜欢通过nvm管理
1. 终端输入`nvm list available`查看目前可以安装的版本
2. 使用命令`nvm install [version]`下载node，如`nvm install 20.0.0`
3. 使用命令`nvm list`查看目前下载的版本，前面有星号的就是当前使用的版本
4. 使用命令`nvm use [version]`安装node，如`nvm use 20.0.0`
5. 使用命令`node -v`和`npm -v`检查是否成功安装
## hexo
使用命令`npm install -g hexo-cli`安装hexo，如果不成功，到文档中查一下版本对应的问题
## 生成博客文件夹
创建一个空文件夹，作为博客的存放地点，在该文件下执行`hexo init`，即可生成博客的基本文件
根目录下的`_config.yml`是站点的配置文件，后续会经常用到
## 启动页面
顺序执行`hexo g`、`hexo s`两个命令，并访问localhost:4000端口，即可打开页面

## 配置部署信息
1. 创建一个名为`[站点名字].github.io`的仓库。
2. 在`_config.yml`中找到deploy项，将其修改为上一步的`github仓库地址`，因为大概率只有一个人写，branch就无所谓了
```yml
deploy:
  type: git
  repo: [github仓库地址]
  branch: master
```

## 部署到GitHub上
使用`hexo d`命令，推送到GitHub上。在GitHub仓库的`setting->page`可以找到访问地址。
# 基本命令
所有命令都是在博客根目录下执行的！
## hexo init
创建一个空文件夹，作为博客的存放地点，在该文件下执行`hexo init`，即可生成博客的基本文件
## hexo new 
执行`hexo new [blogName]`命令，创建一个博客页面，其markdown文件在`\source\_posts`下面
## hexo g
根据配置和markdown文件生成网页文件
## hexo clean
清空之前`hexo g`命令生成的文件
## hexo s
静态部署，在本机4000端口启动服务
## hexo d
根据配置信息部署到GitHub/gitee上
# 主题配置

此处以[next](https://github.com/theme-next/hexo-theme-next)主题为例，介绍配置主题的步骤
## 下载主题
在GitHub等平台扒到喜欢的主题，将其放在`\themes\`目录下，如
```bash
git clone https://github.com/theme-next/hexo-theme-next themes/next
```
## 配置信息
在站点的`_config.yml`中将`theme`改成刚刚下载的主题文件名
如
```bash
theme: next
```
## 刷新站点
```bash
hexo clean
hexo g
hexo s
```
### 需要注意的点
主题文件夹下里也会有一个`_config.yml`，那个一般是配置主题里的东西（比方说next还有四种主题可以切换），不要跟站点的`_config.yml`混淆了

# markdown文件书写
文件头中主要需要配置的是title和tags，多打tags也可以帮助博客文件分类。其实我记得还有另一个参数就是搞文件分类的，去年用过，现在忘了
```
title: hexo搭建与基本操作
date: 2023-10-20 15:11:53
tags:
    - hexo
```
# 多设备同步建议
多建一个仓库，专门同步post文件，不然每次同步整个库要等很久

