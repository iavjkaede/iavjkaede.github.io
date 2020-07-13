---
title: Linux子系统（WSL）Debian安装Nodejs
date: 2019年7月4日
categories: 
	- Deploy
tags: 
	- WSL
	- Npm
	- Nodejs
	- Debian
cover: https://image.zsver.com/2020/05/23/9c88d021ce36d.jpg
---

## 环境简介

### 系统环境

Windows 10 LTSC 2019

！*已安装 Nodejs*

！*已开启 WSL*

WSL ：Debian 发行版

### Shell 环境

ZSH

## 安装Nodejs

```bash
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
sudo apt install -y nodejs
```

**相应的版本可在 setup_*10.x*** 处替换，例如 8.x/9.x

## 检查是否安装成功

```bash
node -v
```

## 配置环境变量

因为是在WSL中安装，如果在Win10 中也安装了Nodejs，在WSL中使用npm命令会出现找不到npm的情况，下面说下怎么解决。

### 先看一下npm在哪

```bash
where npm
```

不出意料的情况下会显示以下内容

```bash
/usr/bin/npm
/mnt/c/Program Files/nodejs//npm
```

第一条是npm在WSL中的位置，第二条是npm在Windows中的位置（取决于你安装时选择的位置）

---

### 添加环境变量

因为我用的是zsh，说一下如何添加

```bash
vi ~/.zshrc
```

在 .zshrc 添加一行

```bash
export PATH=$PATH:usr/bin
```

然后用 source 命令刷新一下

```bash
source ~/.zshrc
```

！*如果你使用的是bash可能不需要添加环境变量这一步*

## 参考

[Ubuntu/debian系安装nodejs环境](https://www.jianshu.com/p/a480b99741f6)

[zsh添加环境变量](https://www.jianshu.com/p/48c022928771)
