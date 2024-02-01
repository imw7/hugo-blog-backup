---
title: Docker的安装与使用
tags: [Docker, 常用组件和技巧]
categories: [Docker]
abbrlink: docker
date: 2020-03-04 00:00:00
draft: false
toc: false
---

本文介绍 `Docker` 在 `Ubuntu` 下的安装以及 `Docker` 的基本命令。<!--more-->

### 安装步骤

1.更新`Ubuntu`的`apt`源索引

```bash
sudo apt-get update
```

2.安装包允许`apt`通过`HTTPS`使用仓库

```bash
sudo dpkg --configure -a
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
```

3.添加`Docker`官方`GPG key`

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

4.设置`Docker`稳定版仓库

```bash
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

5.更新`apt`源索引

```bash
sudo apt-get update
```

6.安装最新版本`Docker CE`（社区版）

```bash
sudo apt-get install docker-ce
```

查看安装`Docker`的版本

```bash
docker --version
```

检查`Docker CE` 是否安装正确

```bash
sudo docker run hello-world
```



### 基本命令

```bash
# 启动docker
sudo service docker start

# 停止docker
sudo service docker stop

# 重启docker
sudo service docker restart

# 列出镜像
docker image ls

# 拉取镜像
docker image pull library/hello-world

# 删除镜像
docker image rm 镜像id/镜像ID

# 创建容器
docker run [选项参数] 镜像名 [命令]

# 停止一个已经在运行的容器
docker container stop 容器名或容器id

# 启动一个已经停止的容器
docker container start 容器名或容器id

# kill掉一个已经在运行的容器
docker container kill 容器名或容器id

# 删除容器
docker container rm 容器名或容器id
```
