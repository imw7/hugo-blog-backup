---
title: SSH使用指南
tags: [ssh, 上传, 下载]
categories: [服务器]
date: 2022-01-02T11:36:55+08:00
abbrlink: ssh
draft: false
toc: true
---

本文主要介绍使用 `SSH` 远程连接的一些技术指导，包括免密登录、文件上传下载、连接超时等问题的解决。<!--more-->

## 免密登录

本节介绍如何实现 `ssh` 免密码连接，通过以下两个步骤省去每次连接都要输入密码的麻烦。

### 生成RSA密钥和公钥

第一步，在本机上使用 `ssh-keygen` 命令来生成 `RSA` 密钥和公钥（如果已经生成过，跳过该步骤）。

```bash
$ ssh-keygen -t rsa
```

**注**：`-t` 表示 `type`，就是说要生成 `RSA` 加密的钥匙。`RSA` 也是默认的加密模型，所以也可以只输入 `ssh-keygen`。默认的 `RSA` 长度是 `2048` 位。

生成 `SSH Key` 的过程中会要求你指定一个文件来保存密钥，按 `Enter` 键使用默认的文件就行了。然后需要输入一个密码来加密你的 `SSH Key`。密码至少要 `20` 位长度(可以不输入)。`SSH` 密钥会保存在 `home` 目录下的 `.ssh/id_rsa` 文件中。`SSH` 公钥保存在 `.ssh/id_rsa.pub` 文件中。

```bash
$ ssh-keygen -t rsa                              
Generating public/private rsa key pair.
Enter file in which to save the key (/home/ashley/.ssh/id_rsa): # 指定一个文件来保存密钥，可以不输入
Created directory '/home/ashley/.ssh'. # 默认
Enter passphrase (empty for no passphrase): # 输入一个密码来加密SSH Key，可以不输入
Enter same passphrase again: 
Your identification has been saved in /home/ashley/.ssh/id_rsa
Your public key has been saved in /home/ashley/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:TG+G6bCPWCIAlVWt3IHFmikwwkqxMZys9EI1UiUwX+U ashley@ubuntu
The key's randomart image is:
+---[RSA 3072]----+
|+BBBo+o*.        |
|.X@ + o +        |
|Bo.+ . E..       |
|+. .. *o.+       |
|. .  .. S +      |
| .     + o       |
|  . . o .        |
|   . + o         |
|    . . .        |
+----[SHA256]-----+
```

### 上传SSH公钥

第二步，使用 `ssh-copy-id` 命令将 `SSH` 公钥上传到 `Linux` 服务器。

```bash
$ ssh-copy-id username@remote-server
```

例如：

```bash
$ ssh-copy-id root@120.77.71.67
The authenticity of host '120.77.71.67 (120.77.71.67)' can't be established.
ECDSA key fingerprint is SHA256:XCuu0vBrfDgrFR6FJljCB7UFw/eLnOHGdQ0FOjtmjqE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 2 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@120.77.71.67's password: 

Number of key(s) added: 2

Now try logging into the machine, with:   "ssh 'root@120.77.71.67'"
and check to make sure that only the key(s) you wanted were added.
```

输入远程用户的密码后，`SSH` 公钥就会自动上传了。`SSH` 公钥保存在远程 `Linux` 服务器的 `.ssh/authorized_keys ` 文件中。

上传成功后，再次使用 `ssh` 远程连接服务器，就不用再输入密码了。

```bash
$ ssh root@120.77.71.67        
Last login: Sun Jul  3 00:30:12 2022 from 222.209.72.69

Welcome to Alibaba Cloud Elastic Compute Service !

[root@centos ~]# 
```

## 上传和下载文件

本节介绍如何向远程服务器上传文件，以及如何从远程服务器上下载文件。

在 `Linux` 下一般用 `scp` 或者 `rsync` 命令来通过 `ssh` 传输文件。

**注意**：用户要有目标的响应权限，下载需要有读权限，上传需要有写权限，否则会提示错误：`Permission denied`。

### 下载文件

有两种方法，用 `scp` 和用 `rsync` 命令都可以实现。

#### 用scp

```bash
scp username@servername:/path/filename /var/www/local_dir 
```

以上命令，可以实现将远程文件下载到本地 `local_dir` 目录，例如：

```bash
scp root@192.168.0.101:/var/www/test.txt /var/www/local_dir
```

这条命令实现了把 `192.168.0.101` 上的 `/var/www/test.txt` 的文件下载到 `/var/www/local_dir`（本地目录）。

#### 用rsync

其中 `-P` 用于显示进度：

```bash
rsync -P -e 'ssh -p 12345' username@servername:/path/filename /var/www/local_dir
```

用 `rsync` 命令还可以实现自动重连：

```bash
RC=1
while [[ $RC -ne 0 ]]
do
	rsync -P -e 'ssh -p 12345' username@servername:/path/filename /var/www/local_dir
	RC=$?
done
```

#### 下载目录

命令格式：

```bash
scp -r username@servername:/var/www/remote_dir/ /var/www/local_dir
```

**其中**：`remote_dir` 为远程目录，`local_dir` 为本地目录，例如：

```bash
scp -r root@192.168.0.101:/var/www/test/ var/www/
```

### 上传文件

命令格式：

```bash
scp /path/filename username@servername:/path
```

例如：

```bash
scp /var/www/test.go root@192.168.0.101:/var/www/
```

该命令实现了把本机 `/var/www/test.go` 文件上传到 `192.168.0.101` 服务器上的 `/var/www/` 目录中。

#### 上传目录

命令格式：

```bash
scp -r local_dir username@servername:remote_dir
```

例如：

```bash
scp -r test root@192.168.0.101:/var/www/
```

该命令实现了**把当前目录下的 `test` 目录上传到服务器的 `/var/www/` 目录**。

#### 指定端口

指定端口用 `-P` 参数，注意是**大写的P**，例如：

```bash
scp -P 8000 -r test root@192.168.0.101:/var/www/
```

这里指定 `8000` 端口。

## 连接超时

本节介绍 `ssh` 连接超时的解决方法和步骤。

#### 方法1

修改 `server` 的 `/etc/ssh/sshd_config`，添加下面两个选项：

```bash
$ vi /etc/ssh/sshd_config
#在文件末尾添加
ClientAliveInterval 60 #server每隔60秒发送一次请求给client，然后client响应，从而保持连接
ClientAliveCountMax 3 #server发出请求后，client没有响应次数达到3，就自动断开连接，一般client会响应。
```

添加完成后，重启系统 `ssh` 服务：

```bash
$ service sshd restart
```

### 方法2

修改 `client` 的 `/etc/ssh/ssh_config`，添加下面两个选项：

```bash
ServerAliveInterval 60 #client每隔60秒发送一次请求给server
ServerAliveCountMax 3 #client发出请求后，server没有响应次数达到3，就自动断开连接，一般server会响应。
```

### 方法3

在连接时加上参数 `-o ServerAliveInterval=60`，例如：

```bash
$ ssh -o ServerAliveInterval=60 root@120.77.71.67
```

这样就能在连接中保持持久连接。