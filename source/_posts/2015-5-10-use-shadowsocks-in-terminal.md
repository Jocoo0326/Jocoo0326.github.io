---
layout: post
title: Use shadowsocks in terminal
date: 2015-05-10
categories:
- linux
tags:
- shadowsocks
---

在终端中使用shadowsocks
<!-- more -->
环境：Ubuntu-14.04-LTS-x64

### 安装proxychains
``` bash
sudo apt-get install proxychains
```
<!--excerpt-->

### 创建配置文件
``` bash
mkdir ~/.proxychains
cd ./proxychains
vim proxychains.conf
```

### 输入以下内容：
``` bash
strict_chain
proxy_dns
remote_dns_subnet 224
tcp_read_time_out 15000
tcp_connect_time_out 8000
localnet 127.0.0.0/255.0.0.0
quiet_mode
                                                                                 
[ProxyList]
#shadowsocks的本地端口
socks5  127.0.0.1 1080
```

### 使用
直接输入
``` bash
proxychains bash
```

即可使用，当然shadowsocks得开启
