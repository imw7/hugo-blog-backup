---
title: VS Code配置Go语言开发环境
tags: ["VS Code", "开发环境准备"]
categories: ["Go"]
date: 2021-03-16 00:00:00
toc: true
---

`VS Code`是微软开源的一款编辑器，插件系统十分的丰富。本文就介绍了如何使用`VS Code`搭建`Go`语言开发环境。<!--more-->

说在前面的话，`Go`语言是采用`UTF8`编码的，理论上使用任何文本编辑器都能做`Go`语言开发。大家可以根据自己的喜好自行选择。`编辑器/IDE`没有最好只有最适合。

## 下载与安装

`VS Code`官方下载地址：https://code.visualstudio.com/Download

三大主流平台都支持，请根据自己的电脑平台选择对应的安装包。![vscode_home](https://blog.imw7.com/images/Go/install_go_dev/vscode_home.png)双击下载好的安装文件，双击安装即可。

## 安装中文简体插件

点击左侧菜单栏最后一项`管理扩展`，在`搜索框`中输入`chinese` ，选中结果列表第一项，点击`install`安装。

安装完毕后右下角会提示`重启VS Code`，重启之后你的VS Code就显示中文啦！![安装简体中文插件](https://blog.imw7.com/images/Go/install_go_dev/vscode1.gif)`VSCode`主界面介绍：

![vscode_menu](https://blog.imw7.com/images/Go/install_go_dev/vscode_menu1.png)

## 安装Go开发扩展

现在我们要为我们的VS Code编辑器安装`Go`扩展插件，让它支持Go语言开发。![安装go扩展图](https://blog.imw7.com/images/Go/install_go_dev/vscode_plugin1.png)

## 变更编辑器主题

依次点击`设置->颜色主题`，

![vscode_theme](https://blog.imw7.com/images/Go/00_config_VSCode/vscode_theme01.png)

会弹出如下窗口：

![vscode_theme](https://blog.imw7.com/images/Go/00_config_VSCode/vscode_theme02.png)

可以根据自己的喜好选择相应的主题。

## 安装Go语言开发工具包

在座Go语言开发的时候为我们提供诸如代码提示、代码自动补全等功能。

在此之前请先设置`GOPROXY`，打开终端执行以下命令：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

Windows平台按下`Ctrl+Shift+P`，Mac平台按`Command+Shift+P`，这个时候VS Code界面会弹出一个输入框，如下图：

![vscode09](https://blog.imw7.com/images/Go/00_config_VSCode/vscode09.png)

我们在这个输入框中输入`>go:install`，下面会自动搜索相关命令，我们选择`Go:Install/Update Tools`这个命令，按下图选中并会回车执行该命令（或者使用鼠标点击该命令）

![vscode10](https://blog.imw7.com/images/Go/00_config_VSCode/vscode10.png)

在弹出的窗口选中所有，并点击“确定”按钮，进行安装。

![vscode11](https://blog.imw7.com/images/Go/00_config_VSCode/vscode11.png)

然后会弹出如下窗口，开始安装工具：

![vscode12](https://blog.imw7.com/images/Go/00_config_VSCode/vscode12.png)

喝口水，等待所有工具都安装成功，如下图所示:

![vscode14](https://blog.imw7.com/images/Go/00_config_VSCode/vscode14.png)

## 配置VSCode开启自动保存

按下图依次点击 `文件->首选项->设置`，

![vscode15](https://blog.imw7.com/images/Go/00_config_VSCode/vscode15.png)

打开设置页面就能看到自动保存相关配置如下图，可以根据自己的喜好选择自动保存的方式：

![vscode16](https://blog.imw7.com/images/Go/00_config_VSCode/vscode16.png)

## 配置代码片段快捷键

还是按`Ctrl/Command+Shift+P`,按下图输入`>snippets`，选择命令并执行：

![vscode16](https://blog.imw7.com/images/Go/00_config_VSCode/vscode17.png)

然后在弹出的窗口点击选择`go`选项：

![vscode18](https://blog.imw7.com/images/Go/00_config_VSCode/vscode18.png)

然后弹出如下页面：

![vscode19](https://blog.imw7.com/images/Go/00_config_VSCode/vscode19.png)

大家可以简单看下上面的注释，介绍了主要用法：

```js
"这里放个名字":{
    "prefix": "这个是快捷键",
    "body": "这里是按快捷键插入的代码片段",
    "description": "这里放提示信息的描述"
}
```

其中`$0`表示最终光标提留的位置。 举个例子，我这里创建了两个快捷方式，一个是输入`pln`就会在编辑器中插入`fmt.Println()`代码；输入`plf`，就会插入`fmt.Printf("")`代码。

```json
{
	"pln":{
		"prefix": "pln",
		"body":"fmt.Println($0)",
		"description": "println"
	},
	"plf":{
		"prefix": "plf",
		"body": "fmt.Printf(\"$0\\n\")",
		"description": "printf"
	}
}
```

把上面的代码，按下图方式粘贴到配置文件中，保存并关闭配置文件即可。

![vscode20](https://blog.imw7.com/images/Go/00_config_VSCode/vscode20.png)

添加如上配置后，保存。 我们打开一个go文件，测试一下效果：

![demo1](https://blog.imw7.com/images/Go/00_config_VSCode/vscode21.gif)