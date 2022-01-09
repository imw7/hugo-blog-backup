---
title: ssh使用scp或rsync上传和下载文件
tags: [ssh, 上传文件]
categories: [远程调用]
date: 2021-12-31T11:36:54
abbrlink: ssh
toc: true
---

本文主要介绍，如何向远程服务器上传文件，以及如何从远程服务器上下载文件。<!--more-->

在 `Linux` 下一般用 `scp` 或者 `rsync` 命令来通过 `ssh` 传输文件。

**注意**：用户要有目标的响应权限，下载需要有读权限，上传需要有写权限，否则会提示错误：`Permission denied`

## 下载文件

有两种方法，用`scp`和用`rsync`命令都可以实现。

### 用`scp`

```bash
scp username@servername:/path/filename /var/www/local_dir 
```

以上命令，可以实现将远程文件下载到本地`local_dir`目录，例如：

```bash
scp root@192.168.0.101:/var/www/test.txt /var/www/local_dir
```

这条命令实现了把`192.168.0.101`上的 `/var/www/test.txt` 的文件下载到 `/var/www/local_dir`（本地目录）。

### 用 `rsync`

其中`-P`用于显示进度：

```bash
rsync -P -e 'ssh -p 12345' username@servername:/path/filename /var/www/local_dir
```

用`rsync`命令还可以实现自动重连：

```bash
RC=1
while [[ $RC -ne 0 ]]
do
	rsync -P -e 'ssh -p 12345' username@servername:/path/filename /var/www/local_dir
	RC=$?
done
```

## 上传文件

命令格式：

```bash
scp /path/filename username@servername:/path
```

例如：

```bash
scp /var/www/test.go root@192.168.0.101:/var/www/
```

该命令实现了把本机 `/var/www/test.go` 文件上传到 `192.168.0.101` 服务器上的 `/var/www/` 目录中。

## 下载目录

命令格式：

```bash
scp -r username@servername:/var/www/remote_dir/ /var/www/local_dir
```

**其中**：`remote_dir` 为远程目录，`local_dir` 为本地目录，例如：

```bash
scp -r root@192.168.0.101:/var/www/test/ var/www/
```

## 上传目录

命令格式：

```bash
scp -r local_dir username@servername:remote_dir
```

例如：

```bash
scp -r test root@192.168.0.101:/var/www/
```

该命令实现了**把当前目录下的 `test` 目录上传到服务器的 `/var/www/` 目录**。

## 指定端口

指定端口用 `-P` 参数，注意是<!--color:red-->**大写的P**，例如：

```bash
scp -P 8000 -r test root@192.168.0.101:/var/www/
```

这里指定 `8000` 端口。
