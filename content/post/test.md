---
title: "test"
date: 2026-06-18
tags: []
categories: []
---

这是一个测试文档
```bash
# 安装 nginx
[root@nginx ~ 11:14:02]# yum install -y nginx

# 启动 nginx
[root@nginx ~ 11:14:14]#  systemctl enable nginx --now

# 准备主页
[root@nginx ~ 11:20:26]# mv /usr/share/nginx/html/index.html{,.ori}
[root@nginx ~ 11:20:31]# echo Hello World From Nginx > /usr/share/nginx/html/index.html

# 防火墙
[root@nginx ~ 11:20:36]#  firewall-cmd --add-service=http --permanent
[root@nginx ~ 11:20:40]# firewall-cmd --reload

[root@client ~]# curl http://web.ggg.cloud

# windows客户端修改 C:\Windows\System32\drivers\etc\hosts
# Linux或Unix修改 /etc/hosts
# 添加如下记录
10.1.8.10 web.ggg.cloud web