+++
title='将Hugo博客部署到阿里云'
tags=['Hugo', '阿里云']
categories=['Hugo']
date="2023-12-02T16:38:53+08:00"
toc=true

+++

本文主要介绍如何将 `Hugo` 博客网站搭建到阿里云上。<!--more-->

## 本地配置

### 安装Hugo

到 [Hugo Releases](https://github.com/gohugoio/hugo/releases) 下载对应的操作系统版本的 `Hugo` 二进制文件（`hugo` 或 `hugo.exe`）

### 生成站点

使用 `Hugo` 快速生成站点，比如希望生成到 `/path/to/site` 路径：

```bash
hugo new site /path/to/site
```

这样就在 `/path/to/site` 目录里生成了初始站点，进去目录：

```bash
cd /path/to/site
```

站点目录结构：

```bash
  ▸ archetypes/
  ▸ content/
  ▸ layouts/
  ▸ static/
    config.toml
```

### 安装主题

克隆一个主题配置，比如 [maupassant](https://github.com/rujews/maupassant-hugo)。直接克隆到建站目录下。

```bash
cd blog
git clone https://github.com/flysnow-org/maupassant-hugo themes/maupassant
```

然后在博客根目录下修改 `config.toml` 文件。要应用主题，需要修改主题配置选项。

```toml
theme = "maupassant"
```

到这里，本地的建站工作就基本完成了。

## 服务端配置

这一部分主要目的是本地电脑可以通过 `ssh` 等方式连接到云服务器，然后通过命令行方式将博客传到云服务器上。

### 连接服务器

#### Windows

使用 `Xshell` 软件进行连接。或者和其他系统一样采用 `ssh` 连接。

#### 其他系统

使用命令 `ssh user@address` 进行连接。

```bash
ssh root@192.168.1.1
```

### 配置Nginx

#### 安装Nginx

在`Nginx`[官网](nginx.org)查看最新版本或者自己想要的版本，通过`wget`命令下载到本地。

```bash
cd /tmp # 进入临时文件夹
wget http://nginx.org/download/nginx-1.25.3.tar.gz # 下载nginx
tar -zxvf nginx-1.25.3.tar.gz # 解压
cd /tmp/nginx-1.25.3
./configure # 默认安装到/usr/local/nginx
make && make install # 编译安装
```

#### SSL模块安装

安装 `ssl` 证书模块，用于 `https` 请求。

```bash
./configure --with-http_ssl_module
```

#### 配置成系统服务

①创建 `nginx.service` 文件

```bash
vim /usr/lib/systemd/system/nginx.service
```

② `nginx.service` 文件中写入内容

```bash
[Unit]
Description=nginx web service
Documentation=http://nginx.org/en/docs/
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=default.target
```

③改权限

```bash
chmod 755 /usr/lib/systemd/system/nginx.service
```

④文件生效

```bash
systemctl daemon-reload
```

⑤设置开机自启

```bash
systemctl enable nginx.service
```

### 建立博客根目录

将博客的页面文件放在 `/home/www/blog/` 路径下，需要先创建这些文件。

```bash
cd /home
mkdir www
cd www
mkdir blog
```

### 配置nginx路由

建立了博客的根目录后，**需要将 `nginx` 服务器指向这个根目录地址，才能访问到博客页面**，所以需要修改 `nginx` 的配置文件。

文件 `nginx.conf` 是默认的配置文件。**但不采用直接修改 `nginx` 配置文件的方式，而是新建一个文件夹，将自己的配置写在新建的文件夹中**。再利用`include`，在配置文件 `nginx.conf` 中将文件夹引入进来即可。这样若有新的需求时，只需在文件夹中添加新需求的配置文件即可，不会再次修改配置文件 `nginx.conf`，提高效率。

切换到 `/usr/local/nginx/` 目录，在此目录下创建一个文件夹 `vhost`：

```bash
cd /usr/local/nginx
mkdir vhost
cd vhost
```

输入 `vim blog.conf` 新建 `blog.conf` 文件并编辑内容：

```bash
server{
	listen			80;
	root			/home/www/blog;
	server_name 	blog.imw7.com;
	location / {
	}
}
```

其中，`listen ` 代表监听 `80` 端口。`root` 是博客的根目录，页面存放的地址。`server_name` 是服务器名称，填域名 `blog.imw7.com`，将域名和博客的页面根目录绑定。

**打开 `/usr/local/nginx/conf/` 目录下的 `nginx.conf` 文件，添加下面一行代码，将刚才新建的配置文件引入进来。`*.conf` 的意思是将 `vhost` 文件夹下的所有 `.conf` 后缀的文件都引入了进来。注意：要写在 `http{}` 的里面。**

```bash
http {
	include /etc/nginx/vhost/*.conf;
}
```

至此，完成了 `nginx` 路由的配置，将域名和云服务器指定路径进行了绑定。

### Nginx服务器配置SSL证书

#### 前提条件

- 已通过数字证书管理服务控制台签发证书。具体操作，请参见[购买SSL证书](https://help.aliyun.com/zh/ssl-certificate/user-guide/purchase-an-ssl-certificate#task-q3j-zfp-ydb)和[提交证书申请](https://help.aliyun.com/zh/ssl-certificate/user-guide/submit-a-certificate-application#concept-wxz-3xn-yfb)。
- `SSL` 证书绑定的域名已完成 `DNS` 解析，即您的域名与主机 `IP` 地址相互映射。您可以通过 `DNS` 验证证书工具，检测域名 `DNS` 解析是否生效。具体操作，请参见[DNS验证](https://help.aliyun.com/zh/ssl-certificate/user-guide/use-the-certificate-toolkit#section-fyr-11v-9r7)。

#### 步骤一：下载SSL证书

1. 登录[数字证书管理服务控制台](https://yundunnext.console.aliyun.com/?p=cas)。

2. 在左侧导航栏，单击**SSL 证书**。

3. 在**SSL 证书**页面，定位到目标证书，在**操作**列，单击**下载**。

4. 在**服务器类型**为 `Nginx` 的**操作**列，单击**下载**。

   ![ssl1.png](https://blog.imw7.com/images/hugo/ssl1.png)

5. 解压缩已下载的 `SSL` 证书压缩包。

   根据您在提交证书申请时选择的 `CSR` 生成方式，解压缩获得的文件不同，具体如下表所示。

   ![ssl2](https://blog.imw7.com/images/hugo/ssl2.png)

   | **CSR生成方式**                 | **证书压缩包包含的文件**                                     |
   | ------------------------------- | ------------------------------------------------------------ |
   | **系统生成**或**选择已有的CSR** | 证书文件（PEM格式）：Nginx支持安装PEM格式的文件，PEM格式的证书文件是采用Base64编码的文本文件，且包含完整证书链。解压后，该文件以`证书ID_证书绑定域名`命名。私钥文件（KEY格式）：默认以*证书绑定域名*命名。 |
   | **手动填写**                    | 如果您填写的是通过数字证书管理服务控制台创建的CSR，下载后包含的证书文件与**系统生成**的一致。如果您填写的不是通过数字证书管理服务控制台创建的CSR，下载后只包括证书文件（PEM格式），不包含证书密码或私钥文件。您可以通过证书工具，将证书文件和您持有的证书密码或私钥文件转换成所需格式。转换证书格式的具体操作，请参见[证书格式转换](https://help.aliyun.com/zh/ssl-certificate/user-guide/use-the-certificate-toolkit#section-7pl-isf-owk)。 |

一般选择系统生成。

#### 步骤二：在Nginx服务器安装证书

1. 执行以下命令，在 `Nginx` 的 `conf` 目录下创建一个用于存放证书的目录。

   ```shell
   cd /usr/local/nginx/conf  #进入Nginx默认配置文件目录。该目录为手动编译安装Nginx时的默认目录，如果您修改过默认安装目录或使用其他方式安装，请根据实际配置调整。
   mkdir cert  #创建证书目录，命名为cert。
   ```

2. 将证书文件和私钥文件上传到 `Nginx` 服务器的证书目录（`/usr/local/nginx/conf/cert`）。

3. 编辑 `Nginx` 配置文件 `nginx.conf`，修改与证书相关的配置。

   a. 执行以下命令，打开配置文件。

   ```shell
   vim /usr/local/nginx/conf/nginx.conf
   ```

   > **重要**
   >
   > `nginx.conf` 默认保存在 `/usr/local/nginx/conf` 目录下。如果您修改过 `nginx.conf` 的位置，可以执行 `nginx -t`，查看 `nginx` 的配置文件路径，并将 `/usr/local/nginx/conf/nginx.conf` 进行替换。

   b. 在 `nginx.conf` 中定位到 `server` 属性配置。

   ![ssl3](https://blog.imw7.com/images/hugo/ssl3.png)

   c. 删除行首注释符号`#`，并参考如下示例进行修改。

   ```shell
   server {
        #HTTPS的默认访问端口443。
        #如果未在此处配置HTTPS的默认访问端口，可能会造成Nginx无法启动。
        listen 443 ssl;
        
        #填写证书绑定的域名
        server_name blog.imw7.com;
    
        #填写证书文件绝对路径
        ssl_certificate /usr/local/nginx/conf/cert/blog.imw7.com.pem;
        #填写证书私钥文件绝对路径
        ssl_certificate_key /usr/local/nginx/conf/cert/blog.imw7.com.key;
    
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 5m;
   	 
        #自定义设置使用的TLS协议的类型以及加密套件（以下为配置示例，请您自行评估是否需要配置）
        #TLS协议版本越高，HTTPS通信的安全性越高，但是相较于低版本TLS协议，高版本TLS协议对浏览器的兼容性较差。
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
   
        #表示优先使用服务端加密套件。默认开启
        ssl_prefer_server_ciphers on;
    
    
       location / {
              root  /home/www/blog;
              index index.html index.htm;
       }
   }
   ```

   d. **可选**：设置 `HTTP` 请求自动跳转 `HTTPS`。

   如果您希望所有的 `HTTP` 访问自动跳转到 `HTTPS` 页面，可通过 `rewrite` 指令重定向到 `HTTPS`。

   > **重要**
   >
   > 以下代码片段需要放置在 `nginx.conf` 文件中 `server {}` 代码段后面，即设置 `HTTP` 请求自动跳转 `HTTPS` 后，`nginx.conf` 文件中会存在两个`server {}`代码段。

   ```shell
   server {
       listen 80;
       #填写证书绑定的域名
       server_name blog.imw7.com;
       #将所有HTTP请求通过rewrite指令重定向到HTTPS。
       rewrite ^(.*)$ https://$host$1;
       location / {
           index index.html index.htm;
       }
   }
   ```

4. 执行以下命令，重启 `Nginx` 服务。

```bash
cd /usr/local/nginx/sbin  #进入Nginx服务的可执行目录。
./nginx -s reload  #重新载入配置文件。
```

### 安装Git

输入命令：

```bash
yum install git
```

### 建立Git用户

#### 配置用户

输入以下命令添加用户`git`：

```bash
sudo adduser git
```

修改用户权限命令：

```bash
$ sudo chmod 740 /etc/sudoers
```

打开`/etc/sudoers`命令：

```bash
vim /etc/sudoers
```

添加语句：

```bash
git ALL=(ALL) ALL
```

保存退出后，将`sudoers`文件权限改回原样：

```bash
sudo chmod 400 /etc/sudoers
```

#### 设置密码

为新创建的 `git` 用户设置密码：

```bash
sudo passwd git
```

输入两遍密码即可。

### 配置密钥

切换到 `git` 用户，然后在 `~` 目录下创建 `.ssh` 文件夹，命令：

```bash
su git
cd ~
mkdir .ssh
cd .ssh
```

生成公钥密钥文件命令：

```bash
ssh-keygen
```

输入后都按回车即可！

此时在目录下就会有两个文件，分别是`id_rsa`和`id_rsa.pub`，其中`id_rsa.pub`就是公钥文件，复制一份：

```bash
cp id_rsa.pub authorized_keys
```

这样目录下就会有一个`authorized_keys`文件，修改它的权限：

```bash
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

客户端生成密钥：

在本地电脑的命令行工具中输入`ssh-keygen -t rsa`，都回车即可。会在本地电脑的用户文件夹下生成密钥。接着，将本地电脑的`id_rsa.pub`文件的内容拷贝到云端服务器的`authorized_keys`文件末尾（注意换行）。

然后在本地电脑上，打开命令行，使用`ssh`方式等连接云服务器，输入：

```bash
ssh -v git@云服务器的公网IP
```

最后提示`Welcome to Alibaba Cloud Elastic Compute Service!`，说明不用输入密码也登录成功了，即配置密钥成功，以后更新博客部署的时候不用输入密码了。

### 创建Git仓库

注意这里的仓库不是`GitHub`仓库，是自己的服务器上的`Git`仓库。

```bash
cd ~
git init --bare blog.git #创建一个空的仓库
```

### 配置钩子

```bash
vim /home/git/blog.git/hooks/post-receive
```

写入以下文本：

```text
git --work-tree=/home/www/blog --git-dir=/home/git/blog.git checkout -f
```

写完后 `esc` 退出输入模式，接着输入 `:wq` 保存文件并退出。最后配置权限：

```bash
chmod +x ~/blog.git/hooks/post-receive
```

至此，服务端配置基本完成。

## 部署本地到服务端

### 配置文件

把本地网站`config.toml`文件内`baseURL`修改成自己的域名。

```toml
baseURL = "https://blog.imw7.com/"
```

### 域名解析

在阿里云上把自己的域名解析到自己的服务器`IP`上。

### 部署

在本地博客根目录运行 `hugo` 命令后，网站根目录内会生成一个 `public` 文件夹，里面是静态网页文件，把这个`public` 文件夹整个`push`到我们刚刚在服务器端配置的`hugo.git`仓库里面。

```bash
cd public
git init
git add .
git commit -m '部署到阿里云'
git remote add origin git@192.168.1.1:/home/git/hugo.git
git push -u origin master
```

