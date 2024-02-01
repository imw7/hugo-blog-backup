---
title: Git各种命令
tags: [git]
categories: [git]
abbrlink: git_command
date: 2018-04-28 00:00:00
draft: false
toc: true
---

对于程序员来说学会版本控制是必须的技能，本文介绍`Git`的各种命令以及如何将代码托管到`GitHub`上。<!--more-->

## Git的使用

### 创建代码仓库

#### 配置身份

`Git`是分布式版本控制系统，每个机器都必须自拔家门。下面是在命令行或终端中设置自己的用户名和`Email`地址的方法。

```bash
git config --global user.name "username"
git config --global user.email "username@example.com"
```

**注意** `git config`命令的`--global`参数，用了这个参数，表示你这台机器上所有的`Git`仓库都会使用这个配置，当然也可以对某个仓库指定不同的用户名和`Email`地址。

配置好之后可以使用这个命令查看配置。

```bash
git config -l
```

#### 创建仓库代码

进入需要创建的项目的目录下面，然后在这个目录相面输入如下命令。

```bash
git init
```

### 提交本地代码

#### 添加

1. 添加单个文件

```bash
git add 文件名
```

2. 添加整个目录

```bash
git add 目录名
```

3. 添加整个项目所有文件

```bash
git add .
```

#### 提交

完成添加后，需要提交，命令如下。

```bash
git commit -m "first commit"
```

**注意** 在`commit`命令后面一定要通过`-m`参数来加上提交的描述信息，没有描述信息的提交被认为是不合法的。

### 查看修改内容

在项目的根目录下输入如下命令。

```bash
git status
```

查看所有文件的改动内容。

```bash
git diff
```

查看特定文件的改动内容。

``` bash
git diff 文件路径
```

### 撤销未提交的修改

使用`checkout`命令，撤销还未提交的修改内容。

```bash
git checkout 文件路径
```

### 免用户名和密码

命令行输入以下命令：

```bash
git config --global credential.helper store
```

紧接着，使用 `git pull` 或者 `git push` 命令，根据提示输入账号和密码。这时将在 `/home/用户名/` 目录下创建一个`.git-credentials` 文件：

```bash
https://用户名:密码@github.com
```

该文件用于记录账号和密码，这样下次就不用再次输入用户名和密码了。
