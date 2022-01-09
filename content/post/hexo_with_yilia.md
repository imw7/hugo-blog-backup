---
title: Hexo博客搭建以及yilia主题设置
tags: [Hexo, yilia]
abbrlink: hexo_yilia
categories: [Hexo]
date: 2017-05-16 00:00:00
toc: true
---

作为一个技术宅，拥有自己的博客是很酷炫的事情。本文介绍如何从零开始搭建Hexo博客以及Hexo主题yilia的相关设置。<!--more-->

# Hexo博客搭建和yilia主题相关设置

Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 [Markdown](http://daringfireball.net/projects/markdown/)（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。这里的主题我选用的是[yilia](https://github.com/litten/hexo-theme-yilia)，通过几天的摸索，主题相关的设置我也会在稍后介绍。

## Hexo的安装

### 安装前提

安装 Hexo 前需要首先安装下列应用程序：

- [Node.js](http://nodejs.org/) (Node.js 版本需不低于 10.13，建议使用 Node.js 12.0 及以上版本)
- [Git](http://git-scm.com/)

### 安装Git

- Windows：下载并安装 [git](https://git-scm.com/download/win).
- Mac：使用 [Homebrew](http://mxcl.github.com/homebrew/), [MacPorts](http://www.macports.org/) 或者下载 [安装程序](http://sourceforge.net/projects/git-osx-installer/)。
- Linux (Ubuntu, Debian)：`sudo apt-get install git-core`
- Linux (Fedora, Red Hat, CentOS)：`sudo yum install git-core`

> Mac 用户
>
> 如果在编译时可能会遇到问题，请先到 App Store 安装 Xcode，Xcode 完成后，启动并进入 **Preferences -> Download -> Command Line Tools -> Install** 安装命令行工具。

> Windows 用户
>
> 对于中国大陆地区用户，可以前往 [淘宝 Git for Windows 镜像](https://npm.taobao.org/mirrors/git-for-windows/) 下载 git 安装包。

### 安装 Node.js

Node.js 为大多数平台提供了官方的 [安装程序](https://nodejs.org/en/download/)。对于中国大陆地区用户，可以前往 [淘宝 Node.js 镜像](https://npm.taobao.org/mirrors/node) 下载。

其它的安装方法：

- Windows：通过 [nvs](https://github.com/jasongin/nvs/)（推荐）或者[nvm](https://github.com/nvm-sh/nvm) 安装。
- Mac：使用 [Homebrew](https://brew.sh/) 或 [MacPorts](http://www.macports.org/) 安装。
- Linux（DEB/RPM-based）：从 [NodeSource](https://github.com/nodesource/distributions) 安装。
- 其它：使用相应的软件包管理器进行安装，可以参考由 Node.js 提供的 [指导](https://nodejs.org/en/download/package-manager/)

对于 Mac 和 Linux 同样建议使用 nvs 或者 nvm，以避免可能会出现的权限问题。

> Windows 用户
>
> 使用 Node.js 官方安装程序时，请确保勾选 **Add to PATH** 选项（默认已勾选）

> For Mac / Linux 用户
>
> 如果在尝试安装 Hexo 的过程中出现 `EACCES` 权限错误，请遵循 [由 npmjs 发布的指导](https://docs.npmjs.com/resolving-eacces-permissions-errors-when-installing-packages-globally) 修复该问题。强烈建议 **不要** 使用 root、sudo 等方法覆盖权限

> Linux
>
> If you installed Node.js using Snap, you may need to manually run `npm install` in the target folder when [initializing](https://hexo.io/docs/commands#init) a blog.

### 安装 Hexo

所有必备的应用程序安装完成后，即可使用 npm 安装 Hexo。

```bash
$ npm install -g hexo-cli
```

### 进阶安装和使用

对于熟悉 npm 的进阶用户，可以仅局部安装 `hexo` 包。

```bash
$ npm install hexo
```

安装以后，可以使用以下两种方式执行 Hexo：

1. `npx hexo <command>`

2. 将 Hexo 所在的目录下的 `node_modules` 添加到环境变量之中即可直接使用 `hexo <command>`：

   ```bash
   echo 'PATH="$PATH:./node_modules/.bin"' >> ~/.profile
   ```

好了，Hexo安装和使用介绍完毕。下面以Linux为例，教你如何一步一步搭建个人博客。

## Hexo 搭建博客

首先切换root身份，方便后面的操作。然后创建博客目录，在博客目录中初始化Hexo。具体命令如图所示。

![hexo init](https://imw7.github.io/images/hexo/hexo_init.png)

Hexo 初始化后，可以使用`hexo s`命令启动一个本地服务，查看博客效果。

![hexo start](https://imw7.github.io/images/hexo/hexo_start.png)

打开浏览器，输入 http://localhost:4000 ，查看Hexo默认主题下的博客效果。

![init blog](https://imw7.github.io/images/hexo/init_blog.png)

下面介绍如何新建一篇博客，编辑博客，生成博客。具体操作见图。

![hexo cmd](https://imw7.github.io/images/hexo/hexo_cmd.png)

使用`hexo g`命令生成博客，可以看到具体操作。

![hexo_g](https://imw7.github.io/images/hexo/g_location.png)

完成之后，可以使用 `hexo s`  查看新建的博客状态。

![first blog](https://imw7.github.io/images/hexo/first_blog.png)

这样博客就新建成功了。

## 把博客部署到服务器

我们搭建博客，当然是为了能够随时随地查看。只能在本地运行的博客没有多大意义。那么要如何操作呢？

### 新建GitHub仓库

为了部署博客，需要把代码托管到免费的Git仓库，这样就可以通过输入网址来访问自己的博客了。如下图所示。

![new](https://imw7.github.io/images/hexo/new_repository.png)

***注意：仓库的名称必须是`username.github.io`的格式，否则无法访问。***

以后可以直接输入上面的仓库名进行博客访问。

### 安装插件

想要实现网页访问博客的功能，需要先在博客目录`myblog`下安装一个 `hexo-deployer-git` 插件。具体操作如下。

![install plug-in](https://imw7.github.io/images/hexo/install_plugin.png)

### 修改 _config.yml

想要部署到GitHub上需要进行关键的设置，使用vim修改_config.yml文件。具体操作如下。

![edit config](https://imw7.github.io/images/hexo/edit_config.png)

跳转到文件底部，添加deploy相关信息。

![关键设置](https://imw7.github.io/images/hexo/edit_deployment2.png)

保存之后退出。

### 部署到远端

使用`hexo d`命令将博客部署到GitHub上。

![](https://imw7.github.io/images/hexo/hexo_deploy.png)

输入GitHub用户名和密码，成功部署。

![输入用户名密码](https://imw7.github.io/images/hexo/input_name_pwd.png)

浏览器输入博客地址 https://xuelang201201.github.io，可以直接访问了，说明成功了。

![success](https://imw7.github.io/images/hexo/success.png)



## yilia主题相关设置

### 使用yilia

要想使用yilia主题，需要首先将yilia克隆到本地博客下的`themes/yilia`目录里。

![克隆yilia到本地](https://imw7.github.io/images/hexo/clone_yilia.png)

然后修改博客根目录下的 _config.yml 文件，方法相同。把Extensions下的theme由原来的landscape修改为yilia。

![修改主题](https://imw7.github.io/images/hexo/change_theme.png)

然后重新部署到远端，出现类似于下面的博客页面说明修改成功了。

![修改主题成功](https://imw7.github.io/images/hexo/blog_with_yilia.png)

### 博客首页不显示全文

如果不做任何操作，博客默认是会显示全文的，数量少影响不大。但是，如果博客数量特别多，而且都是显示全文的情况下，想要快速浏览就不太现实了。这种情况下，就很有必要选择只在首页显示一部分，用户点进去之后再显示全文。

如何让`yilia`主题首页不显示全文，截断文章？很简单，只需要在`markdown`文档中，手动写入下面这条语句即可。

```js
<!-- more -->
```

具体位置，看情况个人选择。语句块位置的上面部分会显示在首页，余下部分不会显示。

![add more](https://imw7.github.io/images/hexo/add.png)

效果如下图，但是会出现一个小问题。

![显示全文和more重复了](https://imw7.github.io/images/hexo/more.png)

`more`的功能和`展开全文`的作用相同，顾名思义即都是显示完整文章的作用。这样就出现了功能重复，最好能够去掉其中一个。我决定去掉`more`，保留更加美观的`展开全文`。那么具体该如何操作呢？

### 首页隐藏more

要想隐藏`markdown`文本中的`more`，需要进行如下操作：

首先进入 `yourlog/themes/yilia/layout/_partial` 目录，在其中找到`article.ejs`文件，使用`vim`或其他工具打开，找到 `article-more-a`语句块，在其中加入`style="display: none"`即可。代码如下：

``` js
<div class="article-entry" itemprop="articleBody">
    ...
    <a class="article-more-a" style="display: none" href="%- url_for(post.path) %>#more"><%= theme.excerpt_link %> >></a> 
```

添加完成后，保存退出。回到blog根目录，先后执行`hexo clean` 和 `hexo d` 命令。浏览器打开自己的blog网页，会发现已经发生了变化。

![clean](https://imw7.github.io/images/hexo/clean.png)

### 翻页设置

经过设置，博客每页显示10篇文章缩略。难免会出现翻页现象，原主题yilia默认的分页设置是`<<Prev`和`Next>>`，如图所示：

![page problem](https://imw7.github.io/images/hexo/page_problem.png)

我想要更改默认设置，使它显示中文的“上一页”与“下一页”。如何设置呢？

进入存放博客的文件夹，依次打开 `/themes/yilia/layout/_partial/` 找到其中的 `archive.ejs`，使用gedit或其他工具打开进行修改。分别修改8，9行和37，38行，如图。

![archive](https://imw7.github.io/images/hexo/archive2.png)

修改完成后的效果如下图。

![prev](https://imw7.github.io/images/hexo/prev.png)

![next](https://imw7.github.io/images/hexo/next.png)

 

从图中可以看出修改后的效果，但是仍然存在问题。首页依然显示了 `<< Prev`，尾页显示了 `Next>>`，我不想让他们继续显示。可以这样设置，打开

目录 `/themes/yilia/layout/_partial/` 找到 `script.ejs`，删除两个`<a>`标签中的`&laquo; Prev`和`Next &raquo;`。也可以将整个`<a>`标签删除。

![script](https://imw7.github.io/images/hexo/script1.png)

修改完成后的效果如图所示。

![fixed prev](https://imw7.github.io/images/hexo/fixed_prev.png)

![fixed next](https://imw7.github.io/images/hexo/fixed_next.png)

![fixed page](https://imw7.github.io/images/hexo/fixed_page.png)

