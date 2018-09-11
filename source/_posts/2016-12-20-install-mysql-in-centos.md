---
layout: post
title: Install mysql in centos7
date: 2016-12-20
categories:
- linux
tags:
- mysql
---

Centos7中安装MySQL默认安装的是mariadb
<!-- more -->
mariadb是MySQL的一个分支，是MySQL创始人Michael Widenius创建的用于避免Oracle闭源风险，被视为Mysql数据库的替代版本；

### installation
使用centos自带的package manager
``` bash
sudo yum install -y mariadb-server
```
使用systemctl命令将mariadb加入开机启动
``` bash
sudo systemctl enable mariadb.service
```
启动mariadb
``` bash
sudo systemctl start mariadb.service
```
查看运行状态
``` bash
sudo systemctl status mariadb.service
```
看到active(running)代表服务运行成功
强化安全设置
``` bash
sudo mysql_secure_installation
```
登陆数据库
``` bash
sudo mysql -u root -p
```
创建数据库
``` sql
CREATE DATABASE IF NOT EXISTS db_oa charset utf8 COLLATE utf8_general_ci;
```
创建用户
``` sql
CREATE USER 'oa-user'@'localhost' IDENTIFIED BY 'password';
```
赋予用户权限
``` sql
GRANT ALL ON db_oa.* TO 'oa-user'@'localhost';
```
赋予用户远程访问的权限
``` sql
GRANT ALL ON db_oa.* TO 'oa-user'@'%' IDENTIFIED BY 'remote-password';
```
本地访问和远程访问可以赋予不同的权限
