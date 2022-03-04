---
title: "Frp内网穿透"
date: 2021-10-10
draft: false
tags: ["frp"]
categories: ["tech"]
description: "基于腾讯云主机实现mac的内网穿透"
summary: ""
---

# 准备工作

1. 一台具有公网IP的设备，我用的是腾讯云的主机
2. 需要进行穿透的内网的设备，我的iMac
3. 内网穿透工具 [frp](https://github.com/fatedier/frp)

# 开始安装

## 服务器安装

下载frp可执行包

```shell
wget https://github.com/fatedier/frp/releases/download/v0.38.0/frp_0.38.0_linux_amd64.tar.gz
```

解压

```shell
tar -zxf frp_0.38.0_linux_amd64.tar.gz
```

进入文件

```shell
cd frp_0.38.0_linux_amd64/
```

修改配置

```shell
vi frps.ini 
```

```text
[common]
bind_port = 7000
token = 123456
```

启动

```shell
./frps -c frps.ini 
```

## Mac端安装

下载frp可执行包

```shell
https://github.com/fatedier/frp/releases/download/v0.38.0/frp_0.38.0_darwin_amd64.tar.gz
```

解压

```shell
tar -zxf frp_0.38.0_darwin_amd64.tar.gz
```

进入文件

```shell
cd frp_0.38.0_darwin_amd64/
```

修改配置

```shell
vim frpc.ini 
```

```text
[common]
server_addr = x.x.x.x # 公网IP
server_port = 7000
token = 123456

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 7022

[vnc]
type = tcp
local_ip = 127.0.0.1
local_port = 5900
remote_port = 7001
use_encryption = true
use_compression = true

[smb]
type = tcp
local_ip = 127.0.0.1
local_port = 445
remote_port = 7002
```

启动

```shell
./frpc -c frpc.ini 
```

## 服务器开放端口

![x5hMc2BlSFI6PDb](https://i.loli.net/2021/11/13/x5hMc2BlSFI6PDb.png)

# 设置自启动

## 服务器端

创建后台启动模版

```shell
vi /etc/systemd/system/frps.service
```

内容如下：

注意修改frp所在的路径

```
[Unit]
Description=frps
After=network.target

[Service]
ExecStart=/root/frp_0.38.0_linux_amd64/frps -c /root/frp_0.38.0_linux_amd64/frps.ini 

[Install]
WantedBy=multi-user.target
```

启动测试

```shell
systemctl start frps.service
```

查看启动状态

```shell
systemctl status frps.service
```

开机自启

```shell
systemctl enable frps.service
```

## Mac端

编辑自启动文件

```shell
touch ~/Library/LaunchAgents/frpc.plist
vim ~/Library/LaunchAgents/frpc.plist
```

frpc.plist 文件内容如下，注意文件中的 frpc 和 frpc.ini 路径，可以将这两个文件移到下方配置文件的路径下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC -//Apple Computer//DTD PLIST 1.0//EN
http://www.apple.com/DTDs/PropertyList-1.0.dtd >
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>frpc</string>
    <key>ProgramArguments</key>
    <array>
     <string>/usr/local/bin/frp_0.38.0_darwin_amd64/frpc</string>
         <string>-c</string>
     <string>/usr/local/bin/frp_0.38.0_darwin_amd64/frpc.ini</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

加载并生效：

```shell
sudo chown root ~/Library/LaunchAgents/frpc.plist
sudo launchctl load -w ~/Library/LaunchAgents/frpc.plist
```

