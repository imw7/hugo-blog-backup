---
title: 'FastDFS'
tags: [fastdfs]
categories: [Go]
date: "2022-01-26T14:01:14+08:00"
toc: true
draft: false
abbrlink: fastdfs
---

本文主要介绍利用 `FastDFS` 上传和下载文件的基本流程以及安装过程。<!--more-->

## 什么是FastDFS

`FastDFS` 是用 `C` 语言编写的一款开源的分布式文件系统。`FastDFS` 为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用 `FastDFS` 很容易搭建一套高性能的文件服务器集群，提供文件上传、下载等服务。

### 基本组成

`FastDFS` 架构包括 `Tracker server` 和 `Storage server`。客户端请求 `Tracker server` 进行文件上传、下载，通过 `Tracker server` 调度最终由 `Storage server` 完成文件上传和下载。

`Tracker server` 作用是负载均衡和调度，通过 `Tracker server` 在文件上传时，可以根据一些方法找到
`Storage server` 提供文件上传服务。可以将 `tracker` 称为`追踪服务器` 或 `调度服务器`。

`Storage server` 作用是文件存储，客户端上传的文件最终存储在 `Storage` 服务器上， `Storage server` 没有实现自己的文件系统，而是利用操作系统的文件系统来管理文件。可以将 `storage` 称为`存储服务器`。

![fdfs](https://blog.imw7.com/images/Go/fdfs/fdfs.png)

服务端两个角色：

* `Tracker`：管理集群。`tracker` 也可以实现集群。每个 `tracker` 节点地位平等。收集 `Storage` 集群的状态。
* `Storage`：实际保存文件。`Storage` 分为多个组，每个组之间保存的文件是不同的。每个组内部可以有多个成员，组成员内部保存的内容是一样的，组成员的地位是一致的，没有主从的概念。

## 文件上传流程

客户端上传文件的流程如图所示：

![上传文件流程](https://blog.imw7.com/images/Go/fdfs/upload.png)

客户端上传文件后存储服务器将文件ID（如：`group1/M00/00/00/eE1HQ2HxEqOAfyMPAAXs_ju6GX8094.jpg`）返回给客户端，此文件ID用于以后访问该文件的索引信息。文件索引信息包括：组名，虚拟磁盘路径，数据两级目录，文件名。

**组名**：`group1`。文件上传后所在的 `storage` 组名称，在文件上传成功后有 `storage` 服务器返回， 需要客户端自行保存。

**虚拟磁盘路径**：`M00`。`storage` 配置的虚拟路径，与磁盘选项 `store_path*` 对应。如果配置了 `store_path0` 则是`M00`，如果配置了 `store_path1` 则是 `M01`，以此类推。

**数据两级目录**：`00/00`。`storage` 服务器在每个虚拟磁盘路径下创建的两级目录，用于存储数据文件。

**文件名**：`eE1HQ2HxEqOAfyMPAAXs_ju6GX8094.jpg`。与文件上传时（`boy.jpg`）不同。是由存储服务器根据特定信息生成，文件名包含：源存储服务器 IP 地址、文件创建时间戳、文件大小、随机数和文件拓展名等信息。

## 文件下载流程

下载文件流程如图：

![下载文件流程](https://blog.imw7.com/images/Go/fdfs/download.png)

## FastDFS安装

### 使用的系统软件

| 名称                   | 说明                            |
| ---------------------- | ------------------------------- |
| `centos`               | `7.x`                           |
| `libfastcommon`        | `FastDFS`分离出的一些公用函数包 |
| `FastDFS`              | `FastDFS`本体                   |
| `fastdfs-nginx-module` | `FastDFS`和`nginx`的关联模块    |
| `nginx`                | `nginx1.20.2`                   |

### 编译环境

CentOS

```bash
$ yum install -y git gcc gcc-c++ make automake autoconf libtool pcre pcre-devel zlib zlib-devel openssl-devel wget vim
```

Debian

```bash
$ apt-get install -y git gcc g++ make automake autoconf libtool pcre2-utils libpcre2-dev zlib1g zlib1g-dev openssl libssh-dev wget vim
```

### 磁盘目录

| 说明                   | 位置                  |
| ---------------------- | --------------------- |
| 所有安装包             | /usr/local/src        |
| 跟踪服务器数据存储位置 | /root/fastdfs/tracker |
| 存储服务器数据存储位置 | /root/fastdfs/storage |

```bash
$ mkdir -p /root/fastdfs/tracker #创建跟踪数据存储目录
$ mkdir -p /root/fastdfs/storage #创建存储数据存储目录
$ cd /usr/local/src #切换到安装目录准备下载安装包
```

### 安装libfastcommon

```bash
$ git clone https://github.com/happyfish100/libfastcommon.git --depth 1
$ cd libfastcommon/
$ sudo ./make.sh && sudo ./make.sh install #编译安装
```

### 安装FastDFS

```bash
$ cd ../ #返回上一级目录
$ git clone https://github.com/happyfish100/fastdfs.git --depth 1
$ cd fastdfs/
$ sudo ./make.sh && sudo ./make.sh install #编译安装
```

#### 配置文件准备

```bash
$ sudo cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf
$ sudo cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf
$ sudo cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf #客户端文件，测试用
$ sudo cp /usr/local/src/fastdfs/conf/http.conf /etc/fdfs/ #供nginx访问使用
$ sudo cp /usr/local/src/fastdfs/conf/mime.types /etc/fdfs/ #供nginx访问使用
```

### 安装fastdfs-nginx-module

```bash
$ cd ../ #返回上一级目录
$ git clone https://github.com/happyfish100/fastdfs-nginx-module.git --depth 1
$ sudo cp /usr/local/src/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs
```

### 安装nginx

```bash
$ wget http://nginx.org/download/nginx-1.20.2.tar.gz #下载nginx压缩包
$ tar -zxvf nginx-1.20.2.tar.gz #解压
$ cd nginx-1.20.2/
#添加fastdfs-nginx-module模块
$ sudo ./configure --add-module=/usr/local/src/fastdfs-nginx-module/src/
$ sudo make && sudo make install #编译安装
```

**注意**：执行`sudo ./configure ...` 时 `Ubuntu` 可能会出现**缺少`PCRE`库**的错误。解决办法只需要安装缺少的库即可：

```bash
$ sudo apt-get install libpcre3 libpcre3-dev
```

## 单机部署

### tracker配置

```bash
#服务器ip为 120.77.71.67
$ vim /etc/fdfs/tracker.conf
#需要修改的内容如下
port=22122 #tracker服务器端口（默认22122，一般不修改）
base_path=/root/fastdfs/tracker #存储日志和数据的根目录
```

### storage配置

```bash
$ vim /etc/fdfs/storage.conf
#需要修改的内容如下
port=23000 #storage服务端口（默认23000，一般不修改）
base_path=/root/fastdfs/storage #数据和日志文件存储根目录
store_path0=/root/fastdfs/storage #第一个存储目录
tracker_server=120.77.71.67:22122 #tracker服务器IP和端口
http.server_port=8888 #http访问文件的端口（默认8888，看情况修改，和nginx中保持一致）
```

### 启动tracker和storage

```bash
$ sudo fdfs_trackerd /etc/fdfs/tracker.conf
$ sudo fdfs_storaged /etc/fdfs/storage.conf
```

### client测试

```bash
$ vim /etc/fdfs/client.conf
#需要修改的内容如下
maxConns=100 #添加这行
base_path=/root/fastdfs/tracker
tracker_server=120.77.71.67:22122 #tracker服务器IP和端口
```

保存后测试，返回 `ID` 表示成功。如：`group1/M00/00/00/xx.jpg`。

```bash
$ fdfs_upload_file /etc/fdfs/client.conf /root/boy.jpg
group1/M00/00/00/wKgDDGLx_SiALJDLAAXs_ju6GX8290.jpg
```

这一步可能出现的问题有：

1. `unknown directive “ngx_fastdfs_module” in /usr/local/nginx/conf/nginx.conf:88`

解决办法：

```bash
$ sudo ./configure --prefix=/usr/local/nginx --sbin-path=/usr/local/nginx/sbin/nginx --conf-path=/usr/local/nginx/conf/nginx.conf --error-log-path=/usr/local/nginx/logs/error.log --http-log-path=/usr/local/nginx/logs/access.log --pid-path=/usr/local/nginx/logs/pid/nginx.pid --lock-path=/usr/local/nginx/logs/lock/subsys/nginx --with-http_ssl_module --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_gzip_static_module --with-http_stub_status_module --with-http_perl_module --with-ld-opt="-Wl,-E" --with-http_image_filter_module --add-module=/usr/local/src/fastdfs-nginx-module/src/
```

2. `./configure: error: the HTTP image filter module requires the GD library. You can either do not enable the module or install the libraries.`

解决办法，安装相应组件即可。

```bash
$ yum install -y gd gd-devel
```

注意：`Ubuntu` 使用 `apt-get -y install libgd-dev`

3. `./configure: error: perl module ExtUtils::Embed is required`

解决办法：

```bash
$ yum -y install perl-devel perl-ExtUtils-Embed
```

### 配置nginx访问

#### 配置mod_fastdfs.conf
```bash
$ vim /etc/fdfs/mod_fastdfs.conf
#需要修改的内容如下
connect_timeout=10
tracker_server=120.77.71.67:22122  #tracker服务器IP和端口
url_have_group_name=true
store_path0=/root/fastdfs/storage
```
#### 配置nginx.conf
```bash
$ vim /usr/local/nginx/conf/nginx.conf
#添加如下配置
server {
    listen       8888;    # 该端口为storage.conf中的http.server_port相同
    server_name  localhost;
    location ~/group[0-9]/ {
        ngx_fastdfs_module;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```
测试下载，用外部浏览器访问刚才已传过的图片，引用返回的ID。

```bash
http://120.77.71.67:8888/group1/M00/00/00/wKgDDGLx_SiALJDLAAXs_ju6GX8290.jpg
#弹出下载单机部署全部跑通
```

这里可能出现`400 Bad Request`的错误，解决办法有两种：

1. 修改`nginx.conf`配置文件，在开头加上一行 `user root;` （实际开发中遇到的问题）
2. 配置`/etc/fdfs/mod_fastdfs.conf`文件，将`url_have_group_name=false`改为`url_have_group_name=true`。即使用组名访问。

修改完成后记得重新加载`nginx`配置文件，使得修改生效。

## 分布式部署

### tracker配置

```bash
#服务器ip为 192.168.52.2,192.168.52.3,192.168.52.4
#建议用ftp下载下来这些文件 本地修改
$ vim /etc/fdfs/tracker.conf
#需要修改的内容如下
port=22122  # tracker服务器端口（默认22122,一般不修改）
base_path=/root/fastdfs/tracker  # 存储日志和数据的根目录
```

### storage配置

```bash
$ vim /etc/fdfs/storage.conf
#需要修改的内容如下
port=23000  # storage服务端口（默认23000,一般不修改）
base_path=/root/fastdfs/storage  # 数据和日志文件存储根目录
store_path0=/root/fastdfs/storage  # 第一个存储目录
tracker_server=192.168.52.2:22122  # 服务器1
tracker_server=192.168.52.3:22122  # 服务器2
tracker_server=192.168.52.4:22122  # 服务器3
http.server_port=8888  # http访问文件的端口(默认8888,看情况修改,和nginx中保持一致)
```

### client测试

```bash
$ vim /etc/fdfs/client.conf
#需要修改的内容如下
base_path=/root/fastdfs/tracker
tracker_server=192.168.52.2:22122  # 服务器1
tracker_server=192.168.52.3:22122  # 服务器2
tracker_server=192.168.52.4:22122  # 服务器3
#保存后测试,返回ID表示成功 如：group1/M00/00/00/xx.jpg
$ fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/boy.jpg
```

### 配置nginx访问

#### 配置mod_fastdfs.conf

```bash
$ vim /etc/fdfs/mod_fastdfs.conf
#需要修改的内容如下
tracker_server=192.168.52.2:22122  # 服务器1
tracker_server=192.168.52.3:22122  # 服务器2
tracker_server=192.168.52.4:22122  # 服务器3
url_have_group_name=true
store_path0=/root/fastdfs/storage
```
#### 配置nginx.conf
```bash
vim /usr/local/nginx/conf/nginx.conf
#添加如下配置
server {
    listen       8888;    ## 该端口为storage.conf中的http.server_port相同
    server_name  localhost;
    location ~/group[0-9]/ {
        ngx_fastdfs_module;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

## 启动

### 防火墙

```bash
#不关闭防火墙的话无法使用
$ systemctl stop firewalld.service #关闭
$ systemctl restart firewalld.service #重启
```

### tracker

```bash
$ /etc/init.d/fdfs_trackerd start #启动tracker服务
$ /etc/init.d/fdfs_trackerd restart #重启动tracker服务
$ /etc/init.d/fdfs_trackerd stop #停止tracker服务
$ chkconfig fdfs_trackerd on #自启动tracker服务
```

### storage

```bash
$ /etc/init.d/fdfs_storaged start #启动storage服务
$ /etc/init.d/fdfs_storaged restart #重动storage服务
$ /etc/init.d/fdfs_storaged stop #停止动storage服务
$ chkconfig fdfs_storaged on #自启动storage服务
```

### nginx

```bash
$ /usr/local/nginx/sbin/nginx #启动nginx
$ /usr/local/nginx/sbin/nginx -s reload #重启nginx
$ /usr/local/nginx/sbin/nginx -s stop #停止nginx
```

### 检测集群

```bash
$ /usr/bin/fdfs_monitor /etc/fdfs/storage.conf
# 会显示会有几台服务器 有3台就会 显示 Storage 1-Storage 3的详细信息
```
