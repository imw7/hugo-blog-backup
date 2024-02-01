---
title: Docker简易指南
tags: [Docker, 常用组件和技巧]
categories: [Docker]
abbrlink: docker_guide
date: 2022-06-22 10:42:36
toc: true
---

本文是 `Docker` 的简易使用指南，包含方方面面的知识，争取通过本文能够对 `Docker` 有个初步的了解。<!--more-->

## Docker安装

这部分以 `CentOS`  和 `Ubuntu` 为例，根据 `Docker` 官网给出的[安装教程](https://docs.docker.com/engine/install/centos/)进行整理和总结。

### 安装步骤

#### CentOS

##### 确定你是CentOS7及以上版本

```bash
cat /etc/redhat-release # centos
```

##### 卸载旧版本

```bash
sudo yum remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-engine
```

##### yum安装gcc相关

```bash
# 前提：CentOS能上外网
sudo yum -y install gcc
sudo yum -y install gcc-c++
```

##### 安装需要的软件包

```bash
sudo yum install -y yum-utils
```

##### 设置stable镜像仓库

因为防火墙的原因，中国大陆地区需要设置可靠的镜像仓库，以阿里云为例。
```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

##### 更新yum软件包索引

```bash
sudo yum makecache fast
```

##### 安装Docker CE

```bash
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

##### 启动Docker

```bash
sudo systemctl start docker
```

##### 测试

```bash
sudo docker version
sudo docker run hello-world
```

##### 卸载Docker引擎

1. 卸载 `Docker Engine`、`CLI`、`Containerd` 和 `Docker Compose` 软件包：

```bash
sudo yum remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
```

2. 主机上的映像、容器、卷或自定义配置文件不会自动删除。要删除所有映像、容器和卷：

```bash
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```
最后，必须手动删除编辑过的配置文件。

#### Ubuntu

##### 确定是Ubuntu20.04及以上版本

```bash
cat /etc/os-release # ubuntu
```

##### 卸载旧版本

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

##### 设置apt仓库

第一次安装`docker`，需要先设置`apt`仓库。

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

##### 安装最新版

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

##### 测试

```bash
sudo docker version
sudo docker run hello-world
```

##### 卸载

1. 卸载 `Docker Engine`、`CLI`、`Containerd` 和 `Docker Compose` 软件包：

````bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
````

2. 主机上的映像、容器、卷或自定义配置文件不会自动删除。要删除所有映像、容器和卷：

```bash
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

最后，必须手动删除编辑过的配置文件。

#### Docker免sudo权限

安装完 `Docker`，每次必须用 `sudo` 权限操作，比较麻烦。将 `Docker` 加入 `sudo` 用户组，即可默认 `sudo` 权限运行，而不用 `sudo`。具体步骤如下：

1. 添加 `docker group`
```bash
sudo groupadd docker
```
2. 将当前用户添加到 `docker` 组
```bash
sudo gpasswd -a ${USER} docker
```
3. 重启 `docker` 服务
```bash
systemctl restart docker
```
4. 刷新 `docker` 组成员
```bash
newgrp - docker
```

这样，再次使用 `docker` 命令，无需添加 `sudo`。

### 阿里云镜像加速

是什么 https://promotion.aliyun.com/ntms/act/kubernetes.html

注册一个属于自己的阿里云账户（可复用支付宝账号）

#### 获得加速器地址连接

##### 登陆阿里云开发者平台

![aliyun_login](https://blog.imw7.com/images/Go/docker/aliyun_login.png)

##### 点击控制台

![aliyun_console](https://blog.imw7.com/images/Go/docker/aliyun_console.png)

##### 选择容器镜像服务

![aliyun_mirror1](https://blog.imw7.com/images/Go/docker/aliyun_mirror1.png)

##### 获取加速器地址

![aliyun_mirror2](https://blog.imw7.com/images/Go/docker/aliyun_mirror2.png)

#### 粘贴脚本直接执行

##### 安装／升级Docker客户端

推荐安装 `1.10.0` 以上版本的 `Docker` 客户端，参考文档[docker-ce](https://yq.aliyun.com/articles/110806)。

##### 配置镜像加速器

针对 `Docker` 客户端版本大于 `1.10.0` 的用户，可以通过修改 `daemon` 配置文件 `/etc/docker/daemon.json` 来使用加速器。

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://wivds1wh.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
```

#### 重启服务器

```bash
sudo systemctl restart docker
```

## Docker常用命令

### 帮助启动类命令

```bash
# 启动docker
systemctl start docker

# 停止docker
systemctl stop docker

# 重启docker
systemctl restart docker

# 查看docker状态
systemctl status docker

# 开机启动
systemctl enable docker

# 查看docker概要信息
docker info

# 查看docker总体帮助文档
docker --help

# 查看docker命令帮助文档
docker 具体命令 --help
```

### 镜像命令

#### docker images [OPTIONS] [REPOSITORY[:TAG]]

1、列出本地主机上的镜像

```bash
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
ubuntu        latest    27941809078c   2 weeks ago    77.8MB
hello-world   latest    feb5d9fea6a5   9 months ago   13.3kB
```

各个选项说明

| 选项         | 说明             |
| ------------ | :--------------- |
| `REPOSITORY` | 表示镜像的仓库源 |
| `TAG`        | 镜像的标签       |
| `IMAGE ID`   | 镜像`ID`         |
| `CREATED`    | 镜像创建时间     |
| `SIZE`       | 镜像大小         |

注：同一仓库源可以有多个 `TAG` 版本，代表这个仓库源的不同版本，我们使用 `REPOSITORY:TAG` 来定义不同的镜像。如果你不指定一个镜像的版本标签，例如你只使用 `ubuntu`，`Docker` 将默认使用 `ubuntu:latest` 镜像。

2、`OPTIONS` 说明

`-a`：列出本地所有的镜像（含历史影像层）

`-q`：只显示镜像 `ID`

#### docker search [OPTIONS] TERM

1、网站

https://hub.docker.com

2、命令

```bash
docker search [OPTIONS] 镜像名字
```

例子：

```bash
$ docker search redis
NAME  DESCRIPTION        							  STARS     OFFICIAL   AUTOMATED
redis Redis is an open source key-value store that…   11050     [OK]       
...
```

`OPTIONS`说明

`--limit`：只列出`N`个镜像，默认`25`个

```bash
docker search --limit 5 redis
```

#### docker pull [OPTIONS] NAME[:TAG|@DIGEST]

下载镜像命令

```bash
docker pull 镜像名字[:TAG]
```

注：没有 `TAG` 就是最新版，等价于 `docker pull 镜像名字:latest`

#### docker system df

查看镜像/容器/数据卷所占的空间

```bash
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          2         2         77.83MB   0B (0%)
Containers      3         0         49B       49B (100%)
Local Volumes   0         0         0B        0B
Build Cache     0         0         0B        0B
```

#### docker rmi [OPTIONS] IMAGE [IMAGE...]

删除一个或多个镜像

1、删除单个

```bash
docker rmi -f 镜像ID
```

2、删除多个

```bash
docker rmi -f 镜像名1:TAG 镜像名2:TAG
```

3、删除全部

```bash
docker rmi -f $(docker images -qa)
```

面试题：谈谈 `Docker` 虚悬镜像是什么？

仓库名、标签都是 `<none>` 的镜像，俗称虚悬镜像 `dangling image`。

### 容器命令

#### 新建+启动容器

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

1、`OPTIONS` 说明

有些是一个减号，有些是两个减号

`--name="容器新名字"`：为容器指定一个名称

`-d`：后台运行容器并返回容器 `ID`，也即启动守护式容器（后台运行）

**`-i`：以交互模式运行容器，通常与 `-t` 同时使用**

**`-t`：为容器重新分配一个伪输入终端，通常与 `-i` 同时使用**

也即**启动交互式容器（前台有伪终端，等待交互）**

`-P`：**随机**端口映射，大写 `P`

`-p`：**指定**端口映射，小写 `p`

| 参数                            | 说明                                 |
| :------------------------------ | ------------------------------------ |
| `-p hostPort:containerPort`     | 端口映射 `-p 8080:80`                |
| `-p ip:hostPort:containerPort`  | 配置监听地址 `-p 10.0.0.100:8080:80` |
| `-p ip::containerPort`          | 随机分配端口 `-p 10.0.0.100::80`     |
| `-p hostPort:containerPort:udp` | 指定协议 `-p 8080:80:tcp`            |
| `-p 81:80 -p 443:443`           | 指定多个                             |

2、启动交互式容器（前台命令行）

```bash
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
ubuntu        latest    27941809078c   2 weeks ago    77.8MB
hello-world   latest    feb5d9fea6a5   9 months ago   13.3kB
$ docker run -it ubuntu /bin/bash
root@aeea6c8ac772:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 05:04 pts/0    00:00:00 /bin/bash
root         9     1  0 05:04 pts/0    00:00:00 ps -ef
```

使用镜像 `ubuntu:latest` 以交互模式启动一个容器，在容器内执行 `/bin/bash` 命令：

```bash
docker run -it ubuntu /bin/bash
```

**参数说明**：

`-i`：交互式操作

`-t`：终端

`ubuntu`：`Ubuntu`镜像

`/bin/bash`：放在镜像名后的是命令，这里我们希望有个交互式`Shell`，因此用的是 `/bin/bash`

要退出终端，直接输入 `exit`

#### 列出当前所有正在运行的容器

1、命令

```bash
docker ps [OPTIONS]
```

例如：

```bash
$ docker ps
CONTAINER ID  IMAGE   COMMAND      CREATED        STATUS       PORTS  NAMES
0de077b13a8b  ubuntu  "/bin/bash"  6 minutes ago  Up 6 minutes        youthful_ja
```

2、`OPTIONS` 说明

`-a`：列出当前所在**正在运行**的容器+**历史上运行过的**容器

`-l`：显示最近创建的容器

`-n`：显示最近`n`个创建的容器

`-q`：**静默模式，只显示容器编号**

#### 退出容器

两种退出方式

1、`exit`

`run` 进去容器，`exit `退出，**容器停止**

2、`ctrl+p+q`

`run` 进去容器，`ctrl+p+q` 退出，**容器不停止**

#### 启动已停止运行的容器

```bash
docker start 容器ID或者容器名
```

#### 重启容器

```bash
docker restart 容器ID或者容器名
```

#### 停止容器

```bash
docker stop 容器ID或者容器名
```

#### 强制停止容器

```bash
docker kill 容器ID或者容器名
```

#### 删除已停止的容器

```bash
docker rm 容器ID
```

*危险操作*：一次性删除多个容器实例

```bash
docker rm -f $(docker ps -a -q)
```

或者

```bash
docker ps -a -q | xargs docker rm
```

#### 重要

有镜像才能创建容器，这是根本前提（下载一个 `Redis:6.0.8` 镜像演示）

##### 启动守护式容器（后台服务器）

在大部分的场景下，我们希望docker的服务是在后台运行的，我们可以通过 `-d` 指定容器的后台运行模式。

```bash
docker run -d 容器名
```

使用镜像 `ubuntu:latest` 以后台模式启动一个容器：

```bash
docker run -d ubuntu
```

***问题***：然后 `docker ps -a` 进行查看，**会发现容器已经退出**。

很重要的要说明一点：**`Docker` 容器后台运行，就必须有一个前台进程**。

容器运行的命令如果不是那些**一直挂起的命令**（比如运行 `top`，`tail`），就是会自动退出的。

这个是 `Docker` 的机制问题，比如你的 `Web` 容器，我们以 `Nginx` 为例，正常情况下，我们配置启动服务只需要启动响应的 `service` 即可。例如 `service nginx start`。

但是，这样做，`Nginx` 为后台进程模式运行，就导致 `Docker` 前台没有运行的应用，这样的容器后台启动后，会立即自杀，因为它觉得自己没事可做了。

所以，最佳的解决方案是，**将你要运行的程序以前台进程的形式运行，常见就是命令行模式，表示我还有交互操作，别中断，O(∩_∩)O哈哈~**。

##### Redis前后台启动演示例子

①前台交互式启动

```bash
docker run -it redis:6.0.8
```

②后台守护式启动

```bash
docker run -d redis:6.0.8
```

##### 查看容器日志

```bash
docker logs [OPTIONS] CONTAINER
```

例如：

```bash
$ docker logs 8616a62e0639
1:C 21 Jun 2022 06:41:38.228 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 21 Jun 2022 06:41:38.228 # Redis version=6.0.8, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 21 Jun 2022 06:41:38.228 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
1:M 21 Jun 2022 06:41:38.229 * Running mode=standalone, port=6379.
1:M 21 Jun 2022 06:41:38.229 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 21 Jun 2022 06:41:38.229 # Server initialized
1:M 21 Jun 2022 06:41:38.229 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 21 Jun 2022 06:41:38.229 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo madvise > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled (set to 'madvise' or 'never').
1:M 21 Jun 2022 06:41:38.230 * Ready to accept connections
```

##### 查看容器内运行的进程

```bash
docker top CONTAINER [ps OPTIONS]
```

例如：

```bash
$ docker top 8616a62e0639
UID      PID    PPID   C       STIME       TTY    TIME        CMD
polkitd  26537  26518  0       14:41       ?      00:00:02    redis-server *:6379
```

##### 查看容器内部细节

```bash
docker inspect [OPTIONS] NAME|ID [NAME|ID...]
```

例如：

```bash
$ docker inspect 8616a62e0639
[
    {
        "Id": "8616a62e06394d8d64fe64ec14396b50990f29480e2805198b0948548770a717",
        "Created": "2022-06-21T06:41:37.92007182Z",
        "Path": "docker-entrypoint.sh",
        "Args": [
            "redis-server"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 26537,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2022-06-21T06:41:38.221059032Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:16ecd277293476392b71021cdd585c40ad68f4a7488752eede95928735e39df4",
        "ResolvConfPath": "/var/lib/docker/containers/8616a62e06394d8d64fe64ec14396b50990f29480e2805198b0948548770a717/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/8616a62e06394d8d64fe64ec14396b50990f29480e2805198b0948548770a717/hostname",
        "HostsPath": "/var/lib/docker/containers/8616a62e06394d8d64fe64ec14396b50990f29480e2805198b0948548770a717/hosts",
        "LogPath": "/var/lib/docker/containers/8616a62e06394d8d64fe64ec14396b50990f29480e2805198b0948548770a717/8616a62e06394d8d64fe64ec14396b50990f29480e2805198b0948548770a717-json.log",
        "Name": "/jolly_chatelet",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {},
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "CapAdd": null,
            "CapDrop": null,
            "CgroupnsMode": "host",
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "ConsoleSize": [
                0,
                0
            ],
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": null,
            "BlkioDeviceWriteBps": null,
            "BlkioDeviceReadIOps": null,
            "BlkioDeviceWriteIOps": null,
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "KernelMemory": 0,
            "KernelMemoryTCP": 0,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": false,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/cc2828def4f62e8de866edbe12655f583a11c586723c20c1fea627f3e8709600-init/diff:/var/lib/docker/overlay2/a8249d070a3c825a941d4d134852fc1aff730a85aee0241ffa2f053b4ad43bcd/diff:/var/lib/docker/overlay2/a5ffdcd005eb951283de7754a3340a747094d52585561a20aa0a83b4ab0e148a/diff:/var/lib/docker/overlay2/32ebaf91ac0fb463c3ecb19fa9db791d2ae6a81f31aa7acb3559f7f5ede0ca72/diff:/var/lib/docker/overlay2/2b77a0daa2495fc17f3b3c7a007b8ea6305f7c5f0e7fd7a7ccca33bdaf56eb3b/diff:/var/lib/docker/overlay2/7e9409453d956f1debe0fe36110ec21aaefaee8bd3eff56e93fe05f022b097b3/diff:/var/lib/docker/overlay2/08f5399c5f34f601b70ace4a2220a653f836d55a0edf4b4d17cb28936120ed14/diff",
                "MergedDir": "/var/lib/docker/overlay2/cc2828def4f62e8de866edbe12655f583a11c586723c20c1fea627f3e8709600/merged",
                "UpperDir": "/var/lib/docker/overlay2/cc2828def4f62e8de866edbe12655f583a11c586723c20c1fea627f3e8709600/diff",
                "WorkDir": "/var/lib/docker/overlay2/cc2828def4f62e8de866edbe12655f583a11c586723c20c1fea627f3e8709600/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [
            {
                "Type": "volume",
                "Name": "9b02f82540fec8416f382d3f55496e77fdfcc5ab2b1c3f6e90d380a026b59698",
                "Source": "/var/lib/docker/volumes/9b02f82540fec8416f382d3f55496e77fdfcc5ab2b1c3f6e90d380a026b59698/_data",
                "Destination": "/data",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
        "Config": {
            "Hostname": "8616a62e0639",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "6379/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "GOSU_VERSION=1.12",
                "REDIS_VERSION=6.0.8",
                "REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-6.0.8.tar.gz",
                "REDIS_DOWNLOAD_SHA=04fa1fddc39bd1aecb6739dd5dd73858a3515b427acd1e2947a66dadce868d68"
            ],
            "Cmd": [
                "redis-server"
            ],
            "Image": "redis:6.0.8",
            "Volumes": {
                "/data": {}
            },
            "WorkingDir": "/data",
            "Entrypoint": [
                "docker-entrypoint.sh"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "bf61f2e19a43db9d1d4f9fc56ab2000ed702ed91de4da5e776a07e1e31e539dc",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "6379/tcp": null
            },
            "SandboxKey": "/var/run/docker/netns/bf61f2e19a43",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "4c0e86921bef4eb69f04d66c4748c547f2148450854755c37b17f7c6204c996c",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:02",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "f80509d058b8ad3bbd0032e40431d90ed7197236bf97697680124412c1492bb0",
                    "EndpointID": "4c0e86921bef4eb69f04d66c4748c547f2148450854755c37b17f7c6204c996c",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

##### 进入正在运行的容器并以命令行交互

重新进入正在运行的容器有两种方法：

第一种：

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...] 
```

重新进入正在运行的容器中：

```bash
docker exec -it 8616a62e0639 /bin/bash
```

第二种：

```bash
docker attach [OPTIONS] CONTAINER
```

重新进入正在运行的容器：

```bash
docker attach 8616a62e0639
```

**上述两种方法的区别**

① `attach` 直接进入容器启动命令的终端，不会启动新的进程。用 `exit` 退出，会导致容器的停止。

② `exec` 是在容器中打开新的终端，并且可以启动新的进程。用 `exit` 退出，不会导致容器的停止。

推荐使用 `docker exec` 命令，因为退出容器终端不会导致容器的停止。

##### 用之前的Redis容器实例进入试试

```bash
# 进入redis服务
docker exec -it 容器ID /bin/bash
docker exec -it 容器ID redis-cli
# 一般用-d后台启动程序，再用exec进入对应容器实例	
```

例如：

```bash
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS      NAMES
71fd743e48fa   redis:6.0.8   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   6379/tcp   sleepy_gould
$ docker exec -it 71fd743e48fa /bin/bash
root@71fd743e48fa:/data# exit
exit
$ docker exec -it 71fd743e48fa redis-cli
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> set name eric
OK
127.0.0.1:6379> get name
"eric"
```

##### 从容器内拷贝文件到主机上

`容器-->主机`

```bash
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
```

公式：`docker cp 容器ID:容器内路径 目的地主机路径`

1、在容器创建文件

```bash
$ docker exec -it 71fd743e48fa /bin/bash
root@71fd743e48fa:/data# cd /tmp
root@71fd743e48fa:/tmp# touch a.txt
root@71fd743e48fa:/tmp# ls
a.txt
```

2、将文件复制到主机

```bash
$ docker cp 71fd743e48fa:/tmp/a.txt /root/c.txt
$ ls
c.txt
```

`主机-->容器`

```bash
docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH
```

##### 导入和导出容器

1、`export` 导出容器的内容作为一个 `tar` 归档文件[对应 `import` 命令]

```bash
docker export 容器ID > 文件名.tar
```

在容器端创建文件 `a.txt`，并按 `ctrl+p+q` 退出

```bash
$ docker run -it ubuntu /bin/bash
root@06fa2250f3e6:/# cd
root@06fa2250f3e6:~# ls
root@06fa2250f3e6:~# clear
root@06fa2250f3e6:~# touch a.txt 
root@06fa2250f3e6:~# echo "Hello, world!" > a.txt
root@06fa2250f3e6:~# // ctrl+p+q 退出界面，不退出进程
```

在主机端导出整个容器成 `tar` 包

```bash
$ docker export 06fa2250f3e6 > abcd.tar
$ ls
abcd.tar
```

2、`import` 从 `tar` 包中的内容创建一个新的文件系统再导入为镜像[对应 `export`]

```bash
cat 文件名.tar | docker import - 镜像用户/镜像名:镜像版本号
```


```bash
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED              STATUS              PORTS      NAMES
06fa2250f3e6   ubuntu        "/bin/bash"              About a minute ago   Up About a minute              ecstatic_nobel
71fd743e48fa   redis:6.0.8   "docker-entrypoint.s…"   About an hour ago    Up About an hour    6379/tcp   sleepy_gould
$ docker rm -f 06fa2250f3e6
06fa2250f3e6
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS       PORTS      NAMES
71fd743e48fa   redis:6.0.8   "docker-entrypoint.s…"   2 hours ago   Up 2 hours   6379/tcp   sleepy_gould
$ cat abcd.tar | docker import - ashley/ubuntu:1.0.0
sha256:daa7fecb01e3e80143aa665fbc41abcc683d8531eeae4387598f84ad1a1d5fa9
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS       PORTS      NAMES
71fd743e48fa   redis:6.0.8   "docker-entrypoint.s…"   2 hours ago   Up 2 hours   6379/tcp   sleepy_gould
$ docker images
REPOSITORY      TAG       IMAGE ID       CREATED          SIZE
ashley/ubuntu   1.0.0     daa7fecb01e3   16 seconds ago   77.8MB
ubuntu          latest    27941809078c   2 weeks ago      77.8MB
hello-world     latest    feb5d9fea6a5   9 months ago     13.3kB
redis           6.0.8     16ecd2772934   20 months ago    104MB
$ docker run -it daa7fecb01e3 /bin/bash
root@ae152bad6453:/# cd
root@ae152bad6453:~# ls
a.txt
root@ae152bad6453:~# cat a.txt
Hello, world!
root@ae152bad6453:~# 
```

## Docker镜像

### 是什么

#### 概念

镜像是一种轻量级、可执行的独立软件包，它包含运行某个软件所需的所有内容，我们把应用程序和配置依赖打包好形成一个可交付的运行环境（包括代码、运行时需要的库、环境变量和配置文件等），这个打包好的运行环境就是 `image` 镜像文件。

只有通过这个镜像文件才能生成 `Docker` 容器实例（类似 `Go` 中 `new` 出来一个对象）。

#### 分层的镜像

**镜像是分层的**，以我们的 `pull` 为例，在下载过程中我们可以看到 `Docker` 的镜像好像是在一层一层的下载。

```bash
$ docker pull mysql
Using default tag: latest

72a69066d2fe: Pulling fs layer 
93619dbc5b36: Pull complete 
99da31dd6142: Pull complete 
626033c43d70: Pull complete 
37d5d7efb64e: Extracting [==================================================>]     149B/149B
ac563158d721: Download complete 
d2ba16033dad: Download complete 
688ba7d5c01a: Download complete 
00e060b6d11d: Downloading [===================>                               ]     42MB/105.2MB
1c04857f594f: Download complete 
4d7cfa90e6ea: Download complete 
e0431212d27d: Download complete
```

#### UnionFS（联合文件系统）

`UnionFS` 是一种分层、轻量级并且高性能的文件系统，它支持**对文件系统的修改作为一次提交来一层层的叠加**，同时可以将不同且目录挂载到同一个虚拟文件系统下 「`unite several directories into a single virtual filesystem`」。`Union` 文件系统是 `Docker` 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

特性：一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。

#### Docker镜像加载原理

`Docker` 的镜像实际上由一层一层的文件系统组成，这种层级的文件系统叫 `UnionFS`。

`bootfs`「`boot file system` 」主要包含 `bootloader` 和 `kernel`，`bootloader` 主要是引导加载 `kernel`，`Linux` 刚启动时会加载 `bootfs` 文件系统，在 **`Docker` 镜像的最底层是引导文件系统 `bootfs`**。这一层与我们典型的 `Linux/Unix` 系统是一样的，包含 `boot` 加载器和内核。当 `boot` 加载完成之后整个内核就都在内存中了，此时内存的使用权已由 `bootfs` 转交给内核，此时系统也会卸载 `bootfs`。

`rootfs`「`root file system` 」，在 `bootfs` 之上，包含的就是典型 `Linux` 系统中的 `/dev`，`/proc`，`/bin`，`/etc` 等标准目录和文件。`rootfs` 就是各种不同的操作系统发行版，比如 `Ubuntu`，`CentOS` 等等。

![linux_file_system](https://blog.imw7.com/images/Go/docker/linux_file_system.png)

平时我们安装进虚拟机的 `Ubuntu` 都是好几个G，为什么 `Docker` 这里才 `77.8M`？

```bash
$ docker images ubuntu
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
ubuntu       latest    27941809078c   2 weeks ago   77.8MB
```

对于一个精简的`OS`，`rootfs` 可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用 `Host` 的 `kernel`，自己只需要提供 `rootfs` 就行了。由此可见对于不同的 `Linux` 发行版，`bootfs` 基本是一致的，`rootfs` 会有差别，因此不同的发行版可以公用 `bootfs`。

#### 为什么Docker镜像要采用这种分层结构呢

采用镜像分层最大的一个好处就是共享资源，方便复制迁移。之所以采用这种分层结构为的就是方便复用。

比如说有多个镜像都从相同的 `base` 镜像构建而来，那么 `Docker Host` 只需在磁盘上保存一份 `base` 镜像；同时内存中也只需加载一份 `base` 镜像，就可以为所有容器服务了，而且镜像的每一层都可以共享。

### 重点理解

**Docker镜像层都是只读的，容器层是可写的**。

当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作”容器层”，”容器层”之下的都叫”镜像层”。

![docker_container_image](https://blog.imw7.com/images/Go/docker/docker_container_image.png)

所有对容器的改动，无论添加、删除、还是修改文件都只会发生在容器层钟。只有容器层是可写的，容器层下面的所有镜像层都是只读的。

### Docker镜像commit操作案例

#### docker commit提交容器副本使之成为一个新的镜像

```bash
docker commit -m="提交的描述信息" -a="作者" 容器ID 要创建的目标镜像名:[标签名]
```

#### 案例演示ubuntu安装vim

1、从 `Hub` 上下载 `ubuntu` 镜像到本地并成功运行

2、原始的默认 `ubuntu` 镜像是不带着 `vim` 命令的

3、外网连通的情况下，安装 `vim`

```bash
$ docker run -it ubuntu /bin/bash
root@f840a57094ad:/# vim a.txt
bash: vim: command not found
root@b06b07b5c64f:/# apt-get update
Get:1 http://archive.ubuntu.com/ubuntu jammy InRelease [270 kB]
......                             
Fetched 21.6 MB in 13s (1675 kB/s)  
Reading package lists... Done
root@f840a57094ad:/# apt-get -y install vim
Reading package lists... Done
......
Processing triggers for libc-bin (2.35-0ubuntu3) ...
```

4、安装完成后，`commit` 成自己的新镜像

```bash
$ docker commit -m="vim cmd add ok" -a="ashley" f840a57094ad imw7/myubuntu:1.1
sha256:1e9b0351af1f8fe4a7863a4cfd0b0d1146fdfae61f8ea56d0fcdef9c1ba73f3c
$ docker images
REPOSITORY      TAG       IMAGE ID       CREATED          SIZE
imw7/myubuntu   1.1       1e9b0351af1f   30 seconds ago   180MB
ubuntu          latest    27941809078c   2 weeks ago      77.8MB
```

5、启动新镜像并和原来的对比

```bash
$ docker images
REPOSITORY      TAG       IMAGE ID       CREATED          SIZE
imw7/myubuntu   1.1       1e9b0351af1f   11 minutes ago   180MB
ubuntu          latest    27941809078c   2 weeks ago      77.8MB
$ docker run -it ubuntu /bin/bash
root@ec7727e9390c:/# vim a.txt
bash: vim: command not found
root@ec7727e9390c:/# exit
exit
$ docker run -it 1e9b0351af1f /bin/bash
root@0e3310eed3ba:/# vim a.txt
root@0e3310eed3ba:/# cat a.txt
This is Docker!
```

#### 小总结

`Docker` 中的镜像分层，**支持通过扩展现有镜像，创建新的镜像**。类似 `Java` 继承于一个 `Base` 基础类，自己再按需扩展。

新镜像是从 `base` 镜像一层一层叠加生成的，每安装一个软件，就在现有镜像的基础上增加一层。

![docker_layer](https://blog.imw7.com/images/Go/docker/docker_layer.png)

## 本地镜像发布到阿里云

### 本地镜像发布到阿里云流程

![aliyun_docker_pull_push](https://blog.imw7.com/images/Go/docker/aliyun_docker_pull_push.png)

### 镜像的生成方法

#### 基于当前容器创建一个新的镜像，新功能增强

```bash
docker commit [OPTIONS] 容器ID [REPOSITORY[:TAG]]
```

`OPTIONS` 说明：

`-a`：提交的镜像作者

`-m`：提交时的说明名字

#### Dockerfile

后续章节介绍

### 将本地镜像推送到阿里云

#### 本地镜像素材原型

```bash
$ docker images
REPOSITORY      TAG       IMAGE ID       CREATED          SIZE
imw7/myubuntu   1.1       1e9b0351af1f   11 minutes ago   180MB
```

#### 阿里云开发者平台

https://promotion.aliyun.com/ntms/act/kubernetes.html

#### 创建仓库镜像

1、选择控制台，进入容器镜像服务

![aliyun_console2](https://blog.imw7.com/images/Go/docker/aliyun_console2.png)

![container_image_service](https://blog.imw7.com/images/Go/docker/container_image_service.png)

2、选择个人实例

![personal_instance](https://blog.imw7.com/images/Go/docker/personal_instance.png)

3、命名空间

![namespace](https://blog.imw7.com/images/Go/docker/namespace.png)

4、仓库名称

![image_hub](https://blog.imw7.com/images/Go/docker/image_hub.png)

5、进入管理界面获得脚本

![get_script](https://blog.imw7.com/images/Go/docker/get_script.png)

#### 将镜像推送到阿里云

##### 将镜像推送到阿里云registry

1、管理界面脚本

![script](https://blog.imw7.com/images/Go/docker/script.png)

2、脚本实例

使用 `docker tag` 命令重命名镜像，并将它通过专有网络地址推送至Registry。

```bash
$ docker images
REPOSITORY      TAG       IMAGE ID       CREATED         SIZE
imw7/myubuntu   1.1       1e9b0351af1f   19 hours ago    180MB
$ docker tag 1e9b0351af1f registry.cn-shenzhen.aliyuncs.com/imw7/myubuntu:1.1
```

使用  `docker push` 命令将该镜像推送至远程。

```bash
docker push registry.cn-shenzhen.aliyuncs.com/imw7/myubuntu:1.1
```

### 将阿里云上的镜像下载到本地

```bash
docker pull registry.cn-shenzhen.aliyuncs.com/imw7/myubuntu:1.1
```

## 本地镜像发布到私有库

### 本地镜像发布到私有库流程

![aliyun_docker_pull_push](https://blog.imw7.com/images/Go/docker/aliyun_docker_pull_push.png)

### 是什么

1、官方 `Docker Hub` 地址：https://hub.docker.com/，中国大陆访问太慢了且有被阿里云取代的趋势，不太主流。

2、`Dockerhub`、阿里云这样的公共镜像仓库可能不太方便，涉及公司机密的不可能提供镜像给公网，所以需要创建一个本地私人仓库供给团队使用，基于公司内部项目构建镜像。

#### Docker Registry

`Docker Registry` 是官方提供的工具，可以用于构建私有镜像仓库。

### 将本地镜像推送到私有库

#### 下载镜像Docker Registry

```bash
docker pull registry
```

#### 运行私有库Registry，相当于本地有个私有Docker Hub

```bash
docker run -d -p 5000:5000 -v /root/myregistry/:/tmp/registry --privileged=true registry
```

默认情况下，仓库被创建在容器的 `/var/lib/registry` 目录下，建议自行用容器卷映射，方便于宿主机联调。

#### 案例演示创建一个新镜像，ubuntu安装ifconifg命令

1、从 `Hub` 上下载 `Ubuntu` 镜像到本地并成功运行

```bash
docker pull ubuntu
```

2、原始的 `Ubuntu` 镜像是不带 `ifconfig` 命令的

```bash
$ docker run -it ubuntu /bin/bash
root@fa41e46960f0:/# ifconfig
bash: ifconfig: command not found
```

3、外网连通的情况下，安装 `ifconfig` 命令并测试通过

① `Docker` 容器内执行下述两条命令

```bash
apt-get update
apt-get install net-tools
```

② 测试通过

```bash
root@fa41e46960f0:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.3  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:03  txqueuelen 0  (Ethernet)
        RX packets 8987  bytes 22503087 (22.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8949  bytes 769069 (769.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

按 `ctrl+p+q` 退出，执行后续操作。

4、安装完成后，`commit` 自己的新镜像

```bash
docker commit -m="提交的描述信息" -a="作者" 容器ID 要创建的目标镜像名:[标签名]
```

```bash
docker commit -m="ifconfig cmd add" -a="ashley" fa41e46960f0 imw7/myubuntu:1.2
```

#### curl验证私服库上有什么镜像

```bash
curl -XGET http://宿主机IP地址:5000/v2/_catalog
```

通过 `ifconfig` 可以查看宿主机IP地址

```bash
$ curl -XGET http://172.30.108.127:5000/v2/_catalog
{"repositories":[]} // 空的
```

#### 将新镜像 imw7/myubuntu:1.2 修改符合私服规范的Tag

按照公式：

```bash
docker tag 镜像:Tag Host:Port/Repository:Tag
```

**注**：填写自己的 `host` 主机 `IP` 地址

使用命令`docker tag`将`imw7/myubuntu:1.2`这个镜像修改为`172.30.108.127:5000/imw7/myubuntu:1.2`。

```bash
$ docker tag imw7/myubuntu:1.2 172.30.108.127:5000/imw7/myubuntu:1.2
$ docker images
REPOSITORY                            TAG       IMAGE ID       CREATED          SIZE
172.30.108.127:5000/imw7/myubuntu     1.2       def83a58fbd8   53 minutes ago   114MB
imw7/myubuntu                         1.2       def83a58fbd8   53 minutes ago   114MB
```

#### 修改配置文件使之支持http

由于 `Docker` 默认不允许 `http` 方式推送镜像，通过配置选项来取消这个限制。

```bash
$ cat /etc/docker/daemon.json
{
  "registry-mirrors": ["https://wivds1wh.mirror.aliyuncs.com"]
}
```

使用 `vim` 命令新增内容：`vim /etc/docker/daemon.json`

```bash
{
  "registry-mirrors": ["https://wivds1wh.mirror.aliyuncs.com"],
  "insecure-registries":["172.30.108.127:5000"]
}
```

注：修改后如果不生效，建议重启 `Docker`。

#### push推送到私服库

```bash
docker push 172.30.108.127:5000/imw7/myubuntu:1.2
```

#### curl验证私服库上有什么镜像2

```bash
$ curl -XGET http://172.30.108.127:5000/v2/_catalog
{"repositories":["imw7/myubuntu"]}
```

#### pull到本地并运行

1、删除本地的镜像文件

```bash
$ docker images
REPOSITORY                          TAG   IMAGE ID       CREATED       SIZE
172.30.108.127:5000/imw7/myubuntu   1.2   def83a58fbd8   2 hours ago   114MB
imw7/myubuntu                       1.2   def83a58fbd8   2 hours ago   114MB
...
$ docker rmi -f 172.30.108.127:5000/imw7/myubuntu:1.2
$ docker rmi -f imw7/myubuntu:1.2
```

2、拉取到本地

```bash
docker pull 172.30.108.127:5000/imw7/myubuntu:1.2
```

查看镜像

```bash
$ docker images
REPOSITORY                                        TAG       IMAGE ID       CREATED         SIZE
imw7/myubuntu                                     1.2       def83a58fbd8   2 hours ago     114MB
172.30.108.127:5000/imw7/myubuntu                 1.2       def83a58fbd8   2 hours ago     114MB
```

3、运行

```bash
$ docker run -it 172.30.108.127:5000/imw7/myubuntu:1.2 /bin/bash
root@93472ebdd6c2:/# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.3  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:03  txqueuelen 0  (Ethernet)
        RX packets 7  bytes 586 (586.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## Docker容器数据卷

**注**：容器卷记得加入 `--privileged=true`。

`Docker` 挂载主机目录访问**如果出现 `cannot open directory .: Permission denied`**，怎么解决呢？

解决办法：在挂载目录后多加一个 `--privileged=true` 参数即可。

如果是 `CentOS7` 安全模块会比之前系统版本加强，不安全的会先禁止，所以目录挂载的情况被默认为不安全的行为。

在 `SELinux` 里面挂载目录被禁止掉了，如果要开启，一般使用 `--privileged=true` 命令，扩大容器的权限解决挂载目录没有权限的问题，也即使用该参数，`Container` 内的 `root` 拥有真正的 `root` 权限，否则，`Container` 内的 `root` 只是外部的一个普通用户权限。

### 是什么

卷就是目录或文件，存在于一个或多个容器中，由 `Docker` 挂载到容器，但不属于联合文件系统，因此能够绕过 `Union File System` 提供一些用于持续存储或共享数据的特性。

卷的设计目的就是**数据的持久化**，完全独立于容器的生存周期，因此 `Docker` 不会在容器删除时删除其挂载的数据卷。

1、一句话：有点类似 `Redis` 里面的 `rdb` 和 `aof` 文件

2、将 `Docker` 容器内的数据保存进宿主机的磁盘中

3、运行一个带有容器卷存储功能的容器实例

```bash
docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录 镜像名
```

### 能干嘛

#### 需求描述

将运用与运行的环境打包镜像，`run` 后形成容器实例运行，但是对数据的要求希望是持久化的。

#### 作用

`Docker` 容器产生的数据，如果不备份，那么当容器实例删除后，容器内的数据自然也就没有了。为了能保存数据在 `Docker` 中我们使用卷。

#### 特点

1、数据卷可在容器之间共享或重用数据

2、卷中的更改可以直接实时生效

3、数据卷中的更改不会包含在镜像的更新中

4、数据卷的生命周期一直持续到没有容器使用它为止

### 数据卷案例

#### 宿主vs容器之间映射添加容器卷

##### 直接命令添加

```bash
docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录 镜像名
```

在 `Docker` 中创建命名为 `u1` 的 `Ubuntu` 镜像，并实现文件的映射：

```bash
$ docker run -it --privileged=true -v /tmp/host_data:/tmp/docker_data --name=u1 ubuntu
root@10f4b9049f6f:/# cd /tmp/docker_data/
root@10f4b9049f6f:/tmp/docker_data# ls
root@10f4b9049f6f:/tmp/docker_data# touch dockerin.txt
root@10f4b9049f6f:/tmp/docker_data# ls
dockerin.txt
```

在宿主机的对应目录中，可以**实时**查看到刚刚在 `Docker` 中创建的文件：

```bash
$ cd /tmp/host_data/
$ ls
dockerin.txt
```

同样的，在主机创建文件，也可以实时在 `Docker` 容器中获得对应的文件。

##### 查看数据卷是否挂载成功

可以通过如下命令，查看数据卷是否挂载成功

```bash
docker inspect 容器ID
```

举个例子：

```bash
$ docker ps 
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS          PORTS     NAMES
10f4b9049f6f   ubuntu    "bash"    15 minutes ago   Up 15 minutes             u1
$ docker inspect 10f4b9049f6f
[
    {
        ...
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/tmp/host_data",
                "Destination": "/tmp/docker_data",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
        ...
    }
]
```

通过 `docker inspect` 命令查看 `Mounts` 模块判断是否数据卷挂载成功。

##### 容器和宿主机之间数据共享

首先在宿主机停掉正在运行的容器。

```bash
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS          PORTS     NAMES
10f4b9049f6f   ubuntu    "bash"    29 minutes ago   Up 29 minutes             u1
$ docker stop 10f4b9049f6f
10f4b9049f6f
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

在宿主机的 `/tmp/host_data/` 文件夹下写入文件。

```bash
$ cd /tmp/host_data/
$ ls
$ touch c.txt
$ ls
c.txt  dockerin.txt  hostin.txt
```

然后在宿主机重新启动刚停止的容器。

```bash
$ docker start 10f4b9049f6f
10f4b9049f6f
$ docker exec -it 10f4b9049f6f /bin/bash
root@10f4b9049f6f:/# cd /tmp/docker_data/
root@10f4b9049f6f:/tmp/docker_data# ls
c.txt  dockerin.txt  hostin.txt
```

可以看到在容器停止运行的情况下，在宿主机写入的数据，当容器重新运行后也能够同步。

总结：

1. `Docker` 修改，主机同步获得

2. 主机修改，`Docker` 同步获得

3. `Docker` 容器 `stop`，主机修改，`Docker` 容器重启后也能同步获得

#### 读写规则映射添加说明

##### 读写（默认）

```bash
docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录:rw 镜像名
```

默认就是 `rw`，因此可以省略。

##### 只读

如果要让容器实例内部被限制，只能读取不能写，可以通过 `/容器内目录:ro 镜像名`，实现功能：

```bash
docker run -it --privileged=true -v /宿主机绝对路径目录:/容器内目录:ro 镜像名
```

注：`ro = read only`

```bash
$ docker run -it --privileged=true -v /mydocker/u:/tmp:ro ubuntu
root@b7d1f80b47c5:/# cd /tmp
root@b7d1f80b47c5:/tmp# ls
root@b7d1f80b47c5:/tmp# touch c.txt
touch: cannot touch 'c.txt': Read-only file system
```

可以看出，在容器内写入内容失败，因为是只读的文件系统。

此时如果宿主机写入内容，可以同步给容器内，容器可以读取到。

宿主机写入内容：

```bash
$ cd /mydocker/u/
$ ls
$ touch a.txt
$ ls
a.txt
```

容器内可以读到：

```bash
root@b7d1f80b47c5:/tmp# ls
a.txt
```

#### 卷的继承和共享

1、容器1完成和宿主机的映射

容器1：

```bash
$ docker run -it --privileged=true -v /mydocker/u:/tmp --name u1 ubuntu
root@5a8ae43aa393:/# cd /tmp
root@5a8ae43aa393:/tmp# ls
root@5a8ae43aa393:/tmp# touch u1_data.txt
root@5a8ae43aa393:/tmp# ls
u1_data.txt
```

宿主机：

```bash
$ cd /mydocker/u
$ ls
u1_data.txt
```

2、容器2继承容器1的卷规则

```bash
docker run -it --privileged=true --volumes-from 父类 --name u2 ubuntu
```

容器2：

```bash
$ docker run -it --privileged=true --volumes-from u1 --name u2 ubuntu
root@43f70de5eb61:/# cd /tmp
root@43f70de5eb61:/tmp# ls
u1_data.txt
root@43f70de5eb61:/tmp# touch u2_data.txt
root@43f70de5eb61:/tmp# ls
u1_data.txt  u2_data.txt
```

宿主机：

```bash
$ cd /mydocker/u/
$ ls
u1_data.txt  u2_data.txt
```

**注**：停掉容器1，容器2也能和宿主机互联互通。恢复容器1之后，容器2和宿主机在容器1停掉期间产生的数据也能在容器1中同步。

## Docker常规安装简介

### 总体步骤

#### 搜索镜像

```bash
docker search [OPTIONS] TERM
```

#### 拉取镜像

```bash
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

#### 查看镜像

```bash
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

#### 启动镜像

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

##### 服务端口映射

格式：`-p port:port`

```bash
docker run -d -p 6379:6379 redis:6.0.8
```

#### 停止容器

```bash
docker stop [OPTIONS] CONTAINER [CONTAINER...]
```

#### 移除容器

```bash
docker rm [OPTIONS] CONTAINER [CONTAINER...]
```

### 安装Tomcat

#### DockerHub上面查找Tomcat镜像

```bash
$ docker search tomcat
NAME    DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
tomcat  Apache Tomcat is an open source implementati…   3344      [OK]             
...
```

#### 从DockerHub上拉取Tomcat到本地

```bash
docker pull tomcat
```

#### 查看是否有拉取到的Tomcat

```bash
$ docker images tomcat
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
tomcat       latest    fb5657adc892   6 months ago   680MB
```

#### 使用Tomcat镜像创建容器实例（也叫运行镜像）

```bash
docker run -d -p 8080:8080 tomcat
```

`-p`：小写，`主机端口:docker容器端口`

`-P`：大写，**随机分配端口**

`i`：交互

`t`：终端

`d`：后台

```bash
$ docker run -d -p 8080:8080 --name t1 tomcat
788f39f0635eb7925652422217a9c6273f970926c55aef15d17c0c3d9d95ee6a
$ docker ps
CONTAINER ID   IMAGE     COMMAND             CREATED          STATUS          PORTS                                       NAMES
788f39f0635e   tomcat    "catalina.sh run"   12 seconds ago   Up 11 seconds   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   t1
```

#### 访问猫首页

##### 问题

因为新版设置，导致无法访问。

![tomcat1](https://blog.imw7.com/images/Go/docker/tomcat1.png)



##### 解决

1、可能没有映射端口或者没有关闭防火墙

因为是在阿里云云服务器 `ECS` 上的 `Docker` 里安装的 `Tomcat`，所以需要访问 `120.77.71.67:8080`。**需要在安全组中放开8080端口**。

2、把 `webapps.dist` 目录换成 `webapps`

* 先成功启动 `Tomcat`

```bash
$ docker ps
CONTAINER ID   IMAGE     COMMAND             CREATED         STATUS         PORTS                                       NAMES
788f39f0635e   tomcat    "catalina.sh run"   6 minutes ago   Up 6 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   t1
$ docker exec -it t1 /bin/bash
root@788f39f0635e:/usr/local/tomcat# 
```

* 查看 `webapps` 文件夹，为空

```bash
root@788f39f0635e:/usr/local/tomcat# cd webapps
root@788f39f0635e:/usr/local/tomcat/webapps# ls -l
total 0
```

* 把 `webapps.dist` 目录换成 `webapps`

```bash
root@788f39f0635e:/usr/local/tomcat/webapps# cd ..
root@788f39f0635e:/usr/local/tomcat# rm -r webapps
root@788f39f0635e:/usr/local/tomcat# mv webapps.dist webapps
```

修改完成后，可成功访问 `Tomcat` 首页。

![tomcat2](https://blog.imw7.com/images/Go/docker/tomcat2.png)

#### 免修改版说明

```bash
docker pull billygoo/tomcat8-jdk8
docker run -d -p 8080:8080 --name mytomcat8 billygoo/tomcat8-jdk8
```

使用特定版本可以免去最新版的修改过程，简化构建程序。

![tomcat2](https://blog.imw7.com/images/Go/docker/tomcat3.png)

### 安装MySQL

#### DockerHub上查找MySQL镜像

```bash
$ docker search mysql
NAME    DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql   MySQL is a widely used, open-source relation…   12767     [OK] 
...
```

#### 从DockerHub上（阿里云加速器）拉取MySQL5.7镜像到本地

```bash
$ docker pull mysql:5.7
$ docker images mysql:5.7
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
mysql        5.7       c20987f18b13   6 months ago   448MB
```

#### 使用MySQL5.7镜像创建容器（也叫运行镜像）

##### 命令出处

参见 `DockerHub` 官网[MySQL界面](https://hub.docker.com/_/mysql?tab=description)。

##### 简单版

1、使用 `MySQL` 镜像

```bash
$ docker run -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
65ae64574e4e6b301cb30744156d1c46b14e24a6e89f6d01943939da566689e8
$ docker ps
CONTAINER ID   IMAGE       COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
65ae64574e4e   mysql:5.7   "docker-entrypoint.s…"   18 seconds ago   Up 17 seconds   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   objective_satoshi
$ docker exec -it 65ae64574e4e /bin/bash
root@65ae64574e4e:/# mysql -uroot -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.36 MySQL Community Server (GPL)
...
mysql>
```

2、建库表插入数据

```bash
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

mysql> create database db01;
Query OK, 1 row affected (0.00 sec)

mysql> use db01;
Database changed
mysql> create table t1(id int, name varchar(20));
Query OK, 0 rows affected (0.03 sec)

mysql> insert into t1 values(1, 'z3');
Query OK, 1 row affected (0.01 sec)

mysql> select * from t1;
+------+------+
| id   | name |
+------+------+
|    1 | z3   |
+------+------+
1 row in set (0.00 sec)
```

3、外部使用 `Navicat` 连接运行在 `Docker` 上的 `MySQL` 容器实例服务

和下图一样填入正确的 「主机`IP`地址」，「`MySQL` 端口号」，「用户名」和「密码」。点击「测试连接」，如果弹出「连接成功」窗口的，即表明连接是通的。点击 「确定」即可实现 `Navicat` 和 `Docker` 中的 `MySQL`容器的连接。

![navicat_connect](https://blog.imw7.com/images/Go/docker/navicat_connect.png)

**`Ubuntu`上`Navicat`无限试用的方法**：

```bash
sudo rm -rf ~/.config/navicat
sudo rm -rf ~/.config/dconf/user

sudo lsof | grep navicat | grep \\.config
```

**删除以上文件，即可无限试用`Navicat Premium 16`。**

4、问题

① **插入中文报错**

![mysql_problem](https://blog.imw7.com/images/Go/docker/mysql_problem.png)

为什么报错？

**Docker 上默认字符集编码隐患**

通过如下命令查看字符集编码：

```mysql
mysql> SHOW VARIABLES LIKE 'character%';
```

在 `Navicat` 上查看：

![navicat_encoding](https://blog.imw7.com/images/Go/docker/navicat_encoding.png)

在 `Docker` 中的 `MySQL` 命令行查看：

![docker_encoding](https://blog.imw7.com/images/Go/docker/docker_encoding.png)

可以看出，编码有所不同，`Docker` 中文字体默认编码为 `latin1`。需要将它修改为 `utf8` 编码，中文才能正常插入和显示。

② **删除容器后，里面的MySQL数据怎么办**

容器实例一删除，`mysql` 数据也会丢失。

```bash
root@65ae64574e4e:/# exit
exit
$ docker ps
CONTAINER ID   IMAGE       COMMAND                  CREATED             STATUS             PORTS                                                  NAMES
65ae64574e4e   mysql:5.7   "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   objective_satoshi
$ docker rm -f 65ae64574e4e
65ae64574e4e
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

这是很显然的道理，容器被删除后，容器里面的 `MySQL` 也就被删除了，数据自然消失了。

##### 实战版

1、新建 `MySQL` 容器实例

```bash
docker run -d -p 3306:3306 --privileged=true -v /data/mysql/mysql/log:/var/log/mysql -v /data/mysql/mysql/data:/var/lib/mysql -v /data/mysql/mysql/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 --name mysql --restart always mysql:5.7
```

实现 `Docker` 上的 `mysql` 数据同步到宿主机的功能。

2、新建 `my.cnf`

通过容器卷同步给 `mysql` 容器实例，解决中文字符编码问题。

将下面的配置代码写入 `my.cnf` 文件中，可以将中文编码改为 `utf8`。

```ini
[client]
default_character_set=utf8
[mysqld]
collation_server = utf8_general_ci
character_set_server = utf8
```

在宿主机进行如下操作：

```bash
$ cd /data/mysql/conf/
$ ls
$ vim my.cnf
$ cat my.cnf
[client]
default_character_set=utf8
[mysqld]
collation_server = utf8_general_ci
character_set_server = utf8
```

3、重新启动 `MySQL` 容器实例再重新进入并查看字符编码

```bash
$ docker restart mysql
mysql
$ docker ps
CONTAINER ID  IMAGE      COMMAND  CREATED        STATUS         PORTS   NAMES
8044e666f6ff  mysql:5.7  ...      3 minutes ago  Up 8 seconds   ...     mysql
$ docker exec -it 8044e666f6ff /bin/bash
root@8044e666f6ff:/# mysql -uroot -p
Enter password: 
...

mysql> SHOW VARIABLES LIKE 'character%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
```

可以看到字符集编码从 `latin1 `修改为了 `utf8`。

4、再新建库新建表再插入中文测试

![insert_chinese_success](https://blog.imw7.com/images/Go/docker/insert_chinese_success.png)

可以看到成功插入了中文，没有出现错误。

也可在 `Docker` 中的 `mysql` 查询：

```mysql
mysql> select * from t1;
+------+--------+
| id   | name   |
+------+--------+
|    1 | z3     |
|    2 | li4    |
|    3 | 王五   |
+------+--------+
```

成功插入中文，没有问题。

5、结论

```bash
之前的DB无效
    
修改字符集操作+重启MySQL容器实例
    
之后的DB有效，需要新建
```

结论：**`Docker` 安装完 `MySQL` 并 `run` 出容器后，建议先修改完字符集编码，再`新建mysql库-表-插数据`**。

```mysql
mysql> create database db02;
mysql> use db02;
mysql> create table t1(id int, name varchar(20));
```

6、假如将当前容器实例删除，再重新来一次，之前建的 `db01` 还有吗？

**答案是有的**。重新启动一个 `mysql` 容器，进入数据库查看数据，因为挂载了容器数据卷，所以数据会从宿主机中恢复到当前 `Docker` 容器中。

```bash
$ docker run -d -p 3306:3306 --privileged=true -v /data/mysql/mysql/log:/var/log/mysql -v /data/mysql/mysql/data:/var/lib/mysql -v /data/mysql/mysql/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 --name mysql --restart always mysql:5.7
3944d05f9ef3dad023b77dc2626a2ff0ca7db02a71cab875c478a3ff7021cf1b
$ docker ps
CONTAINER ID  IMAGE      COMMAND                 CREATED        STATUS        PORTS                                                 NAMES
3944d05f9ef3  mysql:5.7  "docker-entrypoint.s…"  3 seconds ago  Up 3 seconds  0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp  mysql
$ docker exec -it mysql /bin/bash
root@3944d05f9ef3:/# mysql -uroot -p
Enter password: 
...
mysql> use db01;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from t1;
+------+--------+
| id   | name   |
+------+--------+
|    1 | z3     |
|    2 | li4    |
|    3 | 王五   |
+------+--------+
3 rows in set (0.00 sec)

```

### 安装Redis

#### 拉取镜像

从 `Dokcer Hub` 上（阿里云加速器）拉取 `Redis` 镜像到本地标签为 `6.0.8`。

```bash
docker pull redis:6.0.8
```

#### 入门命令

```bash
$ docker run -d -p 6379:6379 redis:6.0.8
165eb674ce4fa94494958a77e179fe7f8a933b01acaf3897dc8c7e7e9e39eeff
$ docker ps
CONTAINER ID  IMAGE        COMMAND                 CREATED        STATUS        PORTS                                      NAMES
165eb674ce4f  redis:6.0.8  "docker-entrypoint.s…"  5 seconds ago  Up 4 seconds  0.0.0.0:6379->6379/tcp, :::6379->6379/tcp  amazing_chebyshev
$ docker exec -it 165eb674ce4f /bin/bash
root@165eb674ce4f:/data# redis-cli
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> set k1 v1
OK
127.0.0.1:6379> get k1
"v1"
127.0.0.1:6379>
```

#### 命令提醒：容器卷记得加入--privileged=true

`Docker` 挂在主机目录 `Docker` 访问出现`cannot open directory .: Permission denied`。

**解决办法**：在挂载目录后多加一个 `--privileged=true` 参数即可。

#### 新建目录

在 `CentOS` 宿主机下新建目录 `/data/redis`。

```bash
mkdir -p /data/redis
```

#### 拷贝配置文件

将准备好的 `redis.conf` 文件放进宿主机的 `/tmp` 目录，默认出厂的原始 [redis.conf](https://raw.githubusercontent.com/redis/redis/6.0/redis.conf)。

```bash
cp /tmp/redis.conf /data/redis/
```

#### 修改配置文件

在 `/data/redis` 目录下修改 `redis.conf` 文件：

1、开启 `Redis` 验证（可选）

```bash
requirepass foobared
```

2、允许 `Redis` 外地连接

注释掉 `bind 127.0.0.1`

```bash
# bind 127.0.0.1
```

3、`daemonize no`

将 `daemonize yes` 注释掉或设置为 `daemonize no`，因为该配置和 `docker run` 中 `-d` 参数冲突，会导致一直启动失败。

```bash
# daemonize yes
# 或
daemonize no
```

4、开启 `Redis` 数据持久化（可选）

```bash
appendonly yes
```

#### 创建容器

使用 `redis:6.0.8` 镜像创建容器（也叫运行镜像）：

```bash
$ docker run -p 6379:6379 --name myrds --restart always --privileged=true -v /data/redis/redis.conf:/etc/redis/redis.conf -v /data/redis/data:/data -d redis:6.0.8 redis-server /etc/redis/redis.conf
d8a9a2a820193a31675fcacec2ced5e025b5f22dc63634637e4e5a86e2a1b2b8
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
d8a9a2a82019   redis:6.0.8   "docker-entrypoint.s…"   24 seconds ago   Up 23 seconds   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   myrds
```

#### 测试redis-cli连接

```bash
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
d8a9a2a82019   redis:6.0.8   "docker-entrypoint.s…"   24 seconds ago   Up 23 seconds   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   myrds
$ docker exec -it myrds /bin/bash
root@d8a9a2a82019:/data# redis-cli -a password
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> set k1 v1
OK
127.0.0.1:6379> get k1
"v1"
127.0.0.1:6379> 
```

#### 请证明Docker启动使用了自己指定的配置文件

##### 修改前

```bash
$ docker exec -it myrds /bin/bash
root@d8a9a2a82019:/data# redis-cli -a password
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> select 15
OK
```

这里用的配置文件，数据库默认是 `16` 个。

##### 修改后

**注**：修改后记得重启服务。

```bash
# 宿主机/app/redis/redis.conf
# Set the number of databases. The default database is DB 0, you can select
# a different one on a per-connection basis using SELECT <dbid> where
# dbid is a number between 0 and 'databases'-1
databases 10
```

宿主机的修改会同步给 `Docker` 容器里面的配置。

```bash
$ docker restart myrds
myrds
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS         PORTS                                       NAMES
d8a9a2a82019   redis:6.0.8   "docker-entrypoint.s…"   23 minutes ago   Up 7 seconds   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   myrds
$ docker exec -it myrds /bin/bash
root@d8a9a2a82019:/data# redis-cli -a password
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> get k1
"v1"
127.0.0.1:6379> select 15
(error) ERR DB index is out of range
```

### 安装Nginx

见`Docker` 进阶指南 [Portainer](https://blog.imw7.com/post/docker_advanced/#%E7%99%BB%E9%99%86%E5%B9%B6%E6%BC%94%E7%A4%BA%E4%BB%8B%E7%BB%8D%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C%E6%A1%88%E4%BE%8B)。

