---
layout: post
title: Building emacs with msys2
date: 2016-09-06
categories:
- tools
tags:
- emacs
---

emacs官网下载的Windows版本缺少很多功能，比如不能显示jpeg、png等图片，于是打算自己编译emacs
<!-- more -->

## msys2
msys2在msys基础上整合了Cygwin(posix抽象层)和mingw-w64，既提供了windows上unix环境，又提供了使用gnu工具链开发原生Windows应用的能力，值得一提的是，msys2使用archlinux上的package manager--pacman，为包管理提供了强大的支持和便利。

### Install msys2 and setup environment
1. 到官网下载msys2安装包, 选择64位 [msys2官网](https://www.msys2.org/)
2. 傻瓜式安装
3. 编辑C:\msys64\etc\pacman.d\mirrorlist.mingw64文件，覆盖以下内容
```
##
## 64-bit Mingw-w64 repository mirrorlist
##

## Primary
## msys2.org
Server = http://repo.msys2.org/mingw/x86_64/
Server = https://sourceforge.net/projects/msys2/files/REPOS/MINGW/x86_64/
Server = http://www2.futureware.at/~nickoe/msys2-mirror/mingw/x86_64/
Server = https://mirror.yandex.ru/mirrors/msys2/mingw/x86_64/
```
4. 运行msys2 mingw 64-bit，更新pacman
```
pacman -Syu
```
5. 安装基础开发包和图片库
```
pacman -S base-devel mingw-w64-x86_64-toolchain \
mingw-w64-x86_64-xpm-nox mingw-w64-x86_64-libtiff mingw-w64-x86_64-giflib \
mingw-w64-x86_64-libpng mingw-w64-x86_64-libjpeg-turbo mingw-w64-x86_64-librsvg \
mingw-w64-x86_64-libxml2 mingw-w64-x86_64-gnutls
```

## Download emacs source code and compile
从(Emacs官网ftp)[http://ftp.gnu.org/gnu/emacs/]下载emacs-25.3.tar.xz文件，解压缩，进入目录
```
./autogen.sh PKG_CONFIG_PATH=/mingw64/lib/pkgconfig
./configure --host=x86_64-w64-mingw32 --target=x86_64-w64-mingw32 --build=x86_64-w64-mingw32 \
--with-wide-int --with-jpeg --with-xpm --with-png --with-tiff --with-rsvg \
--with-xml2 --with-gnutls --with-xft \
--without-imagemagick
make
make install prefix=/c/emacs-25.3-msys2
```
最后还要将路径C:\msys64\mingw64\bin加入到path环境变量中，emacs才能加载对应的dll, 运行C:\emacs-25.3-msys2\bin\runemacs.exe
OK, done.