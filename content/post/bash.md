---
title: "```bash"
date: 2026-06-18
categories: []
---

#!/bin/bash
###### CentOS 7 初始化配置 ######
# 1 命令提示符
# 2 关闭 selinux
# 3 关闭防火墙
# 4 配置本地仓库
# 5 安装基础软件包
# 6 配置 ssh
# 1 配置命令提示符和历史命令格式
cat >> /etc/bashrc <<'EOF'
PS1='[\[\e[91m\]\u\[\e[93m\]@\[\e[92;1m\]\h\[\e[0m\] \[\e[94m\]\W\[\e[0m\] \[\e[35m\]\t\[\e[0m\]]\[\e[93m\]\$\[\e[0m\] '
HISTTIMEFORMAT="%F %T "
EOF
# 2 关闭 selinux
sed -i '/^SELINUX=/cSELINUX=disabled' /etc/selinux/config
# 3 关闭防火墙
systemctl disable firewalld --now
# 4 配置仓库
curl -s -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -s -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
# 5 安装基础软件包
yum install -y bash-completion vim open-vm-tools lrzsz unzip rsync sshpass
# 6 配置 ssh
echo 'StrictHostKeyChecking no' >> /etc/ssh/ssh_config
echo 'UseDNS no' >> /etc/ssh/sshd_config
# 7 关机打快照
init 0
```
还要加一个脚本