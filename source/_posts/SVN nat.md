---
title: 内网SVN穿透踩坑
date: 2019-3-3
categories: 
    - Deploy
tags: 
    - Deploy
    - NAT
cover: https://image.zsver.com/2020/05/23/1bd82249672b3.jpg
---
# 记一次SVN 内网穿透代理

## 准备

环境是这样的，内网有一台SVN服务器 S，一台能访问外网的主机A。首先，我没有SVN服务器的管理权限，只能通过主机A去访问 服务器S。现在我需要通过外网去访问内网的SVN服务。

我的思路是，用主机A 反代 服务器S ，然后使用内网穿透服务将主机A暴露在外网中。

所以需要做以下准备

1. 按照反向代理服务
2. 选择内网穿透服务

这里我用nginx 做反向代理，用Natapp 做内网穿透，Natapp 是收费（免费的通道域名会隔段时间变动，不嫌弃麻烦可以用）的且需要实名认证，如果觉得不适用可以有自己服务器的话可以自行搭建内网穿透服务，这里不多说。

说一下我的A主机具体环境

Win 10 1903

WSL Debian 9

我选择用WSL 来做这一切，在这里面跑服务再好不过了。

## 开始

现在开始详细的说一下，这里面有坑，需要留意

### 反向代理



首先安装nginx 

```bash
sudo apt install nginx
```



启用 nginx

```bash
sudo service nginx start
```



配置反代

我们在`/etc/nginx/conf.d` 中建立一个配置文件

```bash
sudo vim svnproxy.conf
```

输入以下内容，我做详细解释

```nginx
server {

        listen 7777;	# nginx 监听的端口号，反代的服务通过此端口向外服务
        server_name localhost;
        location /{

        		
                set $fixed_destination $http_destination;
        		
        		# 这个地方很重要，坑在这，如果我们通过 http 协议来访问我们
        		# 的主机A，而主机A 需要通过https协议访问SVN服务器
        		# 那么需要向下面三行这样写
                if ( $http_destination ~* ^http(.*)$ ) {
                	set $fixed_destination https$1;
                }
        		
        		# 反之，如果你通过https协议访问主机A，通过http协议访问 SVN服务器
        		# 需要像这么写，区别很明显，我不再多说
            	#if ( $http_destination ~* ^https(.*)$ ) {
                #	set $fixed_destination http$1;
                #}
        
        		# 不像上面做会出什么问题呢，问题就是，外网通过主机A可以正常进行Update，但是无法Commit，原因大致是http COPY PUT 之类的操作经过反代无法正确执行。
        
                proxy_set_header Destination $fixed_destination;
                proxy_set_header Host $http_host;

        		# 要反代的SVN服务器地址，确保SVN服务能提供Https 访问
                proxy_pass      https://192.168.0.2;
        }
}
```



配置好保存我们重载下nginx

```bash
sudo nginx -t && nginx -s reload
```



可以在本地通过浏览器 访问A 主机的`7777` 端口看一下，用SVN在本地测试以下也Ok

### 穿透

穿透的过程我就不在详细讲了。在Natapp 上开条隧道，web服务用的，然后下载它的程序在WLS启动，网站上都用怎么用哈，代理端口号填写上面配置的端口号就可以了。避免麻烦的话可以买个收费的隧道，价格可以，主要有固定域名。要说一点的是，可以用Supervisor 做个进程守护，然后配置下开机自启（我有写过这个的），完美。



### 最后

有个坑我一定要讲一下，我在主机A上使用SVN服务一切正常，然后反代后通过外网访问SVN，发现重命名的文件没办法提交了。经过一番折磨终于找到了问题，那就是**SVN 仓库文件夹中有中文字符**，很坑，所以SVN仓库里最好不要出现中文路径。

到此，告一段落。



