---
title: Linux子系统（WSL）Debian安装aria2并配置AriaNg
date: 
categories: 
	- Deploy
tags: 
	- WSL
	- Aria2
	- Debian
cover: https://image.zsver.com/2020/05/23/ff71252584474.jpg 

	
---

## 环境简介

### 系统环境

Window 10 LTSC 2019  **开启WSL**

WSL 使用 **Debian 发行版**

### Shell 环境

ZSH

### 将安装的软件版本

aria2    版本 **1.30.0**

AriaNg 版本 **1.1.1**

## 开始

### 安装aria2

首先进行[aria2](https://github.com/aria2/aria2) 的安装

```bash
sudo apt install aria2
```

安装完成输入以下命令查看安装是否成功

```bash
aria2c --version
```

成功输出版本信息后，安装就完成了



### 配置aria2

<span id="aria2_conf">配置aria2</span>

安装是很简单，配置aria2 却是有些棘手的，当然不进行配置也是可以使用的，这里不过多介绍，讲一下我自己的配置。

需要创建三个文件，分别用来保存aria2 的配置、会话和日志。

我这里在用户目录下建立文件夹，其实可以将配置目录放在Windows目录下，这样重新安装

WSL的时候就无需重新创建编写配置文件了。

```bash
cd 
mkdir .aria2
cd .aria2
touch aria2.conf
touch aria2.session
touch aria2.log
```

完成后进行aria2 的配置，这里只记录我自己的配置，详细配置自行搜索查阅。

```ini
# Log Options

log-level=warn
log=/home/$LOGNAME/.aria2/aria2.log

# Running on background
daemon=true

# Download path
# 下载地址，可以放在Windows系统的目录下，比如以下是E盘的download
dir=/mnt/e/Download

# Session Options
input-file=/home/$LOGNAME/.aria2/aria2.session
save-session=/home/$LOGNAME/.aria2/aria2.session
save-session-interval=30

continue=true

# Disk cache
disk-cache=32M

# RPC Options
# 因为只在本地使用，不用rpc安全设置也可以
enable-rpc=true
rpc-allow-origin-all=true
rpc-listen-port=6800	

# 其他更详细的配置自行查阅吧，或者待会我们配置AriaNg就可以具体看可以配置哪些内容了
```

**$LOGNAME** 替换成你用户名

使用配置文件启动 aria2

```bash
aria2c --conf-path /home/$LOGNAME/.aria2/aria2.conf
```

*配置这个地方可能会有坑，如果配置文件写错了，这个命令可能没有任何报错就退出了，或者rpc没有成功开启。执行完这条命令后，在浏览器地址栏输入http://localhost:6800*

*如果出现404而不是未响应，说明rpc是正常开启了的（在没有其他进程占用6800端口的情况下），这里提及的端口号对应配置文件中的端口号`rpc-listen-port`*

另外，我不知道aria2 有没有检查配置文件的命令，我是用笨方法排除错误，把配置文件的`daemon=true` 去掉然后使用配置文件启动aria2c，会提示错误信息的。如果没有提示，可以先把不重要的配置注释掉再次启动，rpc是一定要开启的，其他配置对我来说比较次要。

 

### 配置AriaNg

配置好aria2 可是如果每次下载文件都去执行命令，虽然很hacker，但并不是我想要的。我选择使用[AriaNg](https://github.com/mayswind/AriaNg)来操作aria2 ，AriaNg是一个基于Web的arai2 前端。可以在**github** 的**Release**页面中下载**AllInOne**版本。

[AriaNg Github](https://github.com/mayswind/AriaNg)

AllInOne 版本里面包含一个html文件，这就是我要的，我们直接双击打开在浏览器中查看。

当看到左侧菜单栏最后一条有已连接的绿色标志，表示链接成功。如果一致显示未连接，那可能是配置出了问题，应该仔细看看有没有正确**配置aria2**。

到这时候，我们已经能愉快的使用aria2了，但是，我还想完成两件事情。

- 我希望把AriaNg部署成Web服务，而不是直接在本地去打开html文件（这很不酷）

- 我希望每次开机的时候自动启动aria2c

#### 配置静态文件服务

这里我将使用Nodejs来实现。

有关如何在WSL中安装Nodejs，请参考 {% post_link 'Install nodejs for Debian WSL' %}

安装好Nodejs 后，确定一下npm 命令。

```bash
npm --version
```

我们**建立一个目录存放静态页，将AriaNg（Index.html)文件放在该目录下，并cd 到该目录**

建立一个用于支持web服务的js文件。

```bash
touch server.js
```

编辑该文件输入以下内容并保存。

**server.js**

```js
let express = require("express");
let app = express();

app.use(express.static("./"));

// 端口号自行配置
app.listen(8081,()=>{
        console.log('Server start !');
});
```

emmmm，现在还不能运行，需要装一下**express** 依赖包✔

```bash
npm install express --save
```

启动服务

```bash
node server.js
```



*这几个命令都是在最初建立的目录下完成的*

当你看到终端输出`Server start !`，去浏览器输出http://localhost:8081，看是不是打开了AriaNg👌。

现在打开了静态页服务，不过它还是在终端下执行的，关掉终端后，静态页服务也就关了。

#### 用Supervisor守护进程

首先安装Supervisor

```bash
sudo apt install supervisor
```

在我使用的debian中，安装后会在 `/etc/supervisor` 生成一个**supervisord.conf**文件和一个**conf.d**目录，我们可以在conf.d目录中建立配置。supervisord.conf文件中已经包含了它。*不能保证使用的环境都是一样的，这里只是我自己的环境。*



##### 建立ariaNg 的supervisor配置文件

```bash
cd /etc/supervisor/conf.d
sudo vi ariaNg.conf
```

配置文件内容如下

**ariaNg.conf**

```ini
[program:ariaNg]
command=node server.js
directory=/home/ariaNg	#配置静态页的目录
user=root
autorestart=true
stderr_logfile=/var/log/ariaNg/err.log
stdout_logfile=/var/log/ariaNg/out.log

```

##### 启动supervisor

在启动supervisor前，可能需要建立ariaNg的日志目录

```bash
sudo mkdir /var/log/ariaNg
```



启动supervisor服务

```bash
sudo service supervisor start
```

查看下运行状态

```bash
sudo supervisorctl status
```

看到`aria2Ng                          RUNNING` 然后去浏览器访问一下确定是否成功。

使用supervisorctl 的过程中可能会报**unix:///var/run/supervisor.sock no such file**这个错误。不要慌。

可以先看一下ariaNg 的log文件，看看是哪个地方出了问题。

```bash
cat /var/log/ariaNg/err.log
```

如果是配置的问题，就去修改配置文件。

然后停止supervisor 服务，再次启动。

```bash
sudo service supervisor stop
sudo service supervisor start

sudo supervisorctl status
```

我当时直接重装了supervisor，因为刚开始没卸载干净，还重装了好几遍，😂。

当所有一切都配置好后，我们只需要在浏览器中就能使用aria2 了，不过，我还希望它们能在开机时自动启动。

### 开机自动启动aria2

我可不想每次开机都要打开命令行然后输入命令去启动aria2

```bash
aria2c --conf-path /home/$LOGNAME/.aria2/aria2.conf
sudo service supervisor
```

废话少说，有关如何设置在WSL中开机自动启动服务的方法先去这看一下吧。

{% post_link 'WSL Service Autostart' %}

配置好开机自启脚本后，只需要在init.wsl 中加入以下命令

```bash
/etc/init.d/supervisor $1
aria2c --conf-path=/home/xiaosen/.aria2/aria2.conf
```

搞定👏👏👏



### 最后

这大概就是我安装使用aria2 的整个过程了。

*水平有限，如果某些地方不正确，恳请斧正，感谢！*

### 参考



[Aria2 配置详解](https://www.jianshu.com/p/6adf79d29add)

[用nodejs搭建静态网页服务器居然这么简单](https://blog.csdn.net/wenwushq/article/details/79827718)