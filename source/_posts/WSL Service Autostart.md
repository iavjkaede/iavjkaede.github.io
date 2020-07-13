---
title: Linux子系统(WSL)中的服务开机启动
date: 2019-3-2 12:24
categories: 
	- Deploy
tags: 
	- Debian
	- WSL
cover: https://image.zsver.com/2020/05/23/6a7691490d5dc.jpg
---

## 环境简介

### 系统环境

开了WSL的 **Windows LTSC 2019**

WSL： **Debian 发行版**

---

### Shell 环境

WSL Shell： **ZSH**

---

## 简短截说

这个解决方式是在知乎上看到的，这里仅仅是搬运一下，做个记录。

[WSL 服务自动启动的正确方法](https://zhuanlan.zhihu.com/p/47733615)

### 创建并编辑init.wsl

```bash

sudo vim /etc/init.wsl

```

init.wsl

```bash
#! /bin/sh

/etc/init.d/ssh $1
/etc/init.d/supervisor $1
```

### 设置权限

```bash
sudo chown root:root /etc/init.wsl
sudo chmod +x /etc/init.wsl
```

### 创建VBS脚本

在Windows 中 `Win+R` 运行 `shell :startup`

在弹出的目录中创建`debian.vbs`，如果你用的别的发行版，比如ubuntu18.04，文件名就是`ubuntu1804.vbs`

如果不知道自己的发行版名称可以通过以下命令查看

```powershell
wsl -l
```

！*这条命令是在Ｗidows终端中执行而非在WSL中，在我的系统（LTSC 2019)中这条命令并非有效*

或者可以尝试这个命令

```powershell
wslconfig /l
```

### 编辑VBS脚本

debian.vbs

```vbscript
Set ws = CreateObject("Wscript.Shell")
ws.run "wsl -d debian -u root /etc/init.wsl start", vbhide
```

保存文件后，所有的步骤就都进行完了。

### 2020年7月13日 更

WSL2 中用这种方式有问题，系统启动后不能通过localhost 访问。  
暂时还未找到解决方式。

## 原文链接

[WSL 服务自动启动的正确方法](https://zhuanlan.zhihu.com/p/47733615)
