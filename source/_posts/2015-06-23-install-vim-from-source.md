---
layout: post
title: Install vim from source
date: 2015-06-23
categories:
- linux
tags:
- vim
---
从源码编译安装vim8
<!-- more -->
## 从github下载vim源码
``` bash
git clone https://github.com/vim/vim.git
```

## 安装编译依赖
``` bash
sudo apt-get install ruby ruby-dev libncurses5-dev
```

## 编译
``` bash
cd vim/src
./configure --with-features=huge --enable-gui=gnome2 --enable-pythoninterp=yes
--with-python-config-dir=/usr/lib/python2.7/config-x86_64-linux-gnu
--enable-cscope --enable-rubyinterp=yes --enable-multibyte --with-x
--enable-fail-if-missing
```

## 安装
``` bash
make && sudo make install
```
