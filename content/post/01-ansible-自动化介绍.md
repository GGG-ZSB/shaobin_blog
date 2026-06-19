---
title: "01-Ansible 自动化介绍"
date: 2026-06-19
categories: []
---

## Ansible 自动化介绍
### 什么是 ANSIBLE？
**Ansible is a simple automation language**，通过Playbooks描述和配置IT基础架构。
Ansible可以管理强大的自动化任务，适用于不同的生产环境。同时，Ansible对于新用户来说，也可以很快的上手运用到生产环境。
使用案例：
- OpenStack 搭建和维护
- OpenShift 搭建和维护
- ceph 搭建和维护
### Ansible 特点
- **简单**：Ansible Playbooks 是一个人们非常容易查阅，理解和更改的文本文件，用户不需要具备特定的代码编写技能。
- **功能强大**：可以使用Ansible部署应用，例如配置管理，工作流自动化，网络自动化。还可用于编排整个应用生命周期。
- **无代理**：Ansible 是一个无代理的架构，通过**OpenSSH**或者WinRM连接到hosts，并执行任务，推送小的程序（Ansible modules）到这些主机上。这些程序用于设置系统到预期状态。在Ansible执行完成后，任何之前推送的模块，都会被删除。Ansible可以随时使用，因为被管理主机上不需要配置特定代理。正是因为这点，Ansible 才更加高效和安全。
- **跨平台支持**：可以管理Linux、UNIX、windows 和网络设备。
- **非常准确地描述应用**：Ansible Playbook使用YAML格式描述生产环境。
- **可以通过版本控制管理**：Ansible Playbooks和projects是纯文本格式，可以当作源码存放在版本控制系统中。
- **非常容易与其他系统集成：** HP SA，Puppet，Jenkins，红帽卫星服务器等。
### Ansible 概念和架构
- **NODES**：Ansible架构中有两种计算机类型：
  - **控制节点**，安装有ansible软件的节点。
  - **受管节点**，被ansible管理的Linux系统、Windows系统、网络设备等。
- **INVENTORY**：受管主机清单。
- **PLAYBOOK**：Ansible用户只需要编写playbook，确保主机是预期状态。
  - **每个playbook可以包含多个play。**
  - **每个play会在一组hosts上按顺序执行一系列tasks。**
  - **每个task都执行一个模块**，模块是一个小的代码段（Python，PowerShell，或者其他语言）。Ansible自带几百个模块，执行不同类型自动化任务，例如操作系统文件，安装软件，API调用。Tasks，plays和Playbooks是 idempotent（**幂等的**），在相同的主机上多次安全地执行Playbooks，让主机是正确的状态。如果主机已经是预期状态，则Playbook不会做任何改变。
- **PLUGINS**，添加到Ansible中的代码段，用于扩展Ansible平台。
## Ansible 部署
### 准备实验环境
**准备五台标准化配置的机器：一台controller,四台node{1..4}**
```bash
#标准化配置
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
**准备一个脚本：用于自动设置主机名和IP地址**
```bash
#自动配置ip和主机名
[root@node1 ~ 17:06:10]# cat setip.sh
#!/bin/bash
function usage(){
  echo "Usage: $0 10-14 name"
}
ip_all=$1
ip=${ip_all:1:1}
name=$2
function sethost(){
  if (( ip_all==10 )) ;then
   hostnamectl set-hostname controller.${name}.cloud
   nmcli con mod ens32 ipv4.method manual ipv4.addresses 10.1.8.${ip_all}/24 ipv4.gateway 10.1.8.2 ipv4.dns "223.5.5.5,223.6.6.6" con.autoconnect yes && nmcli con up ens32 &>/dev/null
   echo "10.1.8.${ip_all} controller.${name}.cloud" >> /etc/hosts
  elif (( ip_all > 10 && ip_all <=14 ));then
   hostnamectl set-hostname  node${ip}.${name}.cloud
   nmcli con mod ens32 ipv4.method manual ipv4.addresses 10.1.8.${ip_all}/24 ipv4.gateway 10.1.8.2 ipv4.dns "223.5.5.5,223.6.6.6" con.autoconnect yes && nmcli con up ens32 &>/dev/null
   echo "10.1.8.${ip_all} node${ip}.${name}.cloud" >> /etc/hosts
  else
   usage && exit 1
  fi
}
sethost $1 $2
ip -br a | grep ens32
hostname
```
**配置免密登录**
```bash
#在controller准备一个mianmi.sh脚本来实现
[ggg@contorller ~ 17:11:24]$ cat mianmi.sh
#!/bin/bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa -N ''
for host in 10.1.8.1{0..4}
do
  sshpass -p123456 ssh-copy-id root@$host
done
```
**将weihu脚本上传到/usr/loca/bin/weihu**
```bash
#脚本内容
#!/bin/bash
# 主机清单
HOSTS=("10.1.8.10" "10.1.8.11" "10.1.8.12" "10.1.8.13" "10.1.8.14")
# 默认使用 root 用户维护主机
SSH_USER="root"
# ===================== 功能函数 =====================
# 检查SSH免密登录（成功不显示，失败才显示）
check_ssh_key() {
    for host in "${HOSTS[@]}"; do
        ssh -o BatchMode=yes -o ConnectTimeout=3 "${SSH_USER}@${host}" "echo ok" &>/dev/null
        if [ $? -ne 0 ]; then
            echo -e "\n【错误】主机 $host 未配置 root 免密登录！"
            echo "请先配置 SSH 密钥免密登录后再执行脚本！"
            exit 1
        fi
    done
}
# 批量执行命令
exec_cmd() {
    local cmd="$*"
    echo -e "\n===== 批量执行命令：$cmd ====="
    for host in "${HOSTS[@]}"; do
        ssh -o ConnectTimeout=3 "${SSH_USER}@${host}" "$cmd"
    done
}
# 批量复制文件（支持多文件：src1 src2 ... dest）
copy_file() {
    # 最后一个参数是目标地址，前面全部是源文件/目录
    local dest="${@: -1}"
    local srcs=("${@:1:$#-1}")
    # 校验源文件是否存在
    for src in "${srcs[@]}"; do
        if [ ! -e "$src" ]; then
            echo "【错误】本地文件 $src 不存在！"
            exit 1
        fi
    done
    echo -e "\n===== 批量推送文件 ====="
    for host in "${HOSTS[@]}"; do
        scp -o ConnectTimeout=3 -r "${srcs[@]}" "${SSH_USER}@${host}:${dest}"
    done
}
# 帮助信息
show_help() {
    cat << EOF
===== 批量主机管理脚本 weihu =====
作用：批量管理 10.1.8.10-14 主机（默认 root）
规则：参数数量 **少于2个** 时显示帮助
使用方法：
  1. 批量执行命令
     weihu cmd "需要执行的命令"
  2. 批量复制文件/目录（支持多个源文件）
     weihu copy 源文件1 源文件2 ... 目标路径
  3. 查看帮助
     weihu help
EOF
}
# ===================== 主逻辑 =====================
# 参数少于2个，直接显示帮助
if [ $# -lt 2 ]; then
    show_help
    exit 1
fi
# 帮助命令
if [ "$1" = "help" ]; then
    show_help
    exit 0
fi
# 执行前检查免密（成功静默）
check_ssh_key
# 执行功能
case "$1" in
    cmd)
        shift
        exec_cmd "$*"
        ;;
    copy)
        shift
        copy_file "$@"
        ;;
    *)
        echo "【错误】不支持的命令：$1"
        show_help
        exit 1
        ;;
esac
```
**实验环境 /etc/hosts**
```bash
#创建一个hosts文件
[ggg@contorller ~ 17:15:07]$ cat hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.1.8.10 controller.ggg.cloud controller
10.1.8.11 node1.ggg.cloud node1
10.1.8.12 node2.ggg.cloud node2
10.1.8.13 node3.ggg.cloud node3
10.1.8.14 node4.ggg.cloud node4
```
**将 hosts文件推送到所有主机**
```bash
[ggg@contorller ~ 17:15:09]$ weihu copy hosts /etc/hosts
#查看
[ggg@contorller ~ 17:15:11]$ weihu cmd cat /etc/hosts
.......
```
配置控制节点 ggg 用户使用ggg用户免密登录所有节点，并免提sudo提权执行任何命令。
```bash
# 所有节点添加一个普通用户ggg(已添加)
# 配置所有节点：普通用户免密授权
[ggg@contorller ~ 17:15:21]$ echo 'ggg ALL=(ALL) NOPASSWD:ALL' >  ggg
[ggg@contorller ~ 17:15:24]$ weihu copy ggg /etc/sudoers.d/
#修改远程登录用户
[ggg@contorller ~ 17:15:33]$ sudo -i
[root@contorller ~ 17:15:45]# vim /usr/local/bin/weihu
SSH_USER="ggg"
#配置普通用户免密登录
[ggg@contorller ~ 17:16:11]$ for host in 10.1.8.{10..14}; do sshpass -p123456 ssh-copy-id ggg@$host;done
#验证
[ggg@contorller ~ 17:16:23]$ weihu cmd id
===== 批量执行命令：id =====
uid=1000(ggg) gid=1000(ggg) 组=1000(ggg),10(wheel)
uid=1000(ggg) gid=1000(ggg) 组=1000(ggg),10(wheel)
uid=1000(ggg) gid=1000(ggg) 组=1000(ggg),10(wheel)
uid=1000(ggg) gid=1000(ggg) 组=1000(ggg),10(wheel)
uid=1000(ggg) gid=1000(ggg) 组=1000(ggg),10(wheel)
```
### 控制节点
控制节点用来安装 Ansible 软件的主机节点。控制节点可以是一个或多个，由 ansible 管理的主机不用安装 Ansible。
**提示**：控制节点是Linux或UNIX系统，不支持 Windows 作为控制节点。
安装 ansible
```bash
[ggg@contorller ~ 17:22:10]$ sudo yum install -y ansible
[ggg@contorller ~ 17:22:14]$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/ggg/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /bin/ansible
  python version = 2.7.5 (default, Jun 28 2022, 15:30:04) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
```
# 02-Ansible 基本使用
## Ansible 清单
### Ansible 软件包中文件
```bash
[ggg@contorller ~ 15:32:57]$ rpm -ql ansible
```
- 配置文件目录 	/etc/ansible
- 执行文件目录	/usr/bin
- lib依赖库目录	/usr/lib/python2.7/site-packages/ansible
- 插件                       /usr/share/ansible/plugins
- Help文档目录	/usr/share/doc/ansible
- Man文档目录	/usr/share/man/
### 主机清单
Inventory 定义Ansible将要管理的一批主机。这些主机也可以分配到组中，以进行集中管理 组可以包含子组，主机也可以是多个组的成员。清单还可以设置应用到它所定义的主机和组的变量。
通过以下方式定义主机清单:
- **静态主机清单：**以文本文件的方式来定义。
- **动态主机清单：**使用外部信息提供程序通过**脚本或其他程序**来自动生成。目的是从启动环境中获取主机清单，例如openstack、kubernetes、zabbix等。
### 静态主机清单
主机清单支持多种格式，例如ini、yaml、脚本等。
本笔记使用 **ini** 格式。
#### 最简单的静态清单
受管节点的主机名或IP地址的列表，每行一个。
示例：
```bash
[ggg@contorller ~ 15:36:16]$ vim inventory.example
```
```ini
[webservers]
web1.example.com
web2.example.com
192.168.3.7
[dbservers]
db1.example.com
db2.example.com
192.0.2.42
[eastdc]
web1.example.com
db1.example.com
[westdc]
web2.example.com
db2.example.com
[dc:children]
eastdc
westdc
```
验证主机是否在inventory中
```bash
[ggg@contorller ~ 15:38:35]$ ansible --list-hosts -i inventory.example web1.example.com
  hosts (1):
    web1.example.com
[ggg@contorller ~ 15:39:07]$ ansible --list-hosts -i inventory.example 192.0.2.42
  hosts (1):
    192.0.2.42
```
ansible命令通过--inventory PATHNAME或-i PATHNAME选项在命令行中指定清单文件的位置，其中PATHNAME是所需清单文件的路径。
#### 主机组
还可以将受管节点组织为主机组。通过主机组，更加有效地对一系列系统运行Ansible。
格式：
```ini
[groupname]
hostname
hostip
```
示例：
```ini
#在inventory.example最上面添加一个不属于任何组
app1.example.com
```
验证：
```bash
[ggg@contorller ~ 14:25:21]$ ansible --list-hosts -i inventory webservers
  hosts (3):
    web1.example.com
    web2.example.com
    192.168.3.7
# 注意：192.0.2.43属于dbservers组
[ggg@contorller ~ 15:39:36]$ ansible --list-hosts -i inventory.example dbservers
  hosts (3):
    db1.example.com
    db2.example.com
    192.0.2.42
```
有两个组总是存在的：
- **all**：包含inventory中所有主机。
- **ungrouped**：inventory中列出的，但不属于任何组的主机。
验证：
```bash
[ggg@contorller ~ 15:40:40]$ ansible --list-hosts -i inventory.example all
  hosts (7):
    app1.example.com
    web1.example.com
    web2.example.com
    192.168.3.7
    db1.example.com
    db2.example.com
    192.0.2.42
[ggg@contorller ~ 15:41:13]$ ansible --list-hosts -i inventory.example ungrouped
  hosts (1):
    app1.example.com
```
根据需要，将主机分配在多个组中，例如根据主机的角色、其物理位置以及是否在生产环境中等因素。
```ini
[webservers]
web1.example.com
web2.example.com
192.168.3.7
[dbservers]
db1.example.com
db2.example.com
192.0.2.42
[eastdc]
web1.example.com
db1.example.com
[westdc]
web2.example.com
db2.example.com
```
验证：
```bash
[ggg@contorller ~ 15:42:17]$ ansible --list-hosts -i inventory webservers
  hosts (3):
    web1.example.com
    web2.example.com
    192.168.3.7
[ggg@contorller ~ 15:42:40]$ ansible --list-hosts -i inventory eastdc
  hosts (2):
    web1.example.com
    db1.example.com
```
#### 主机组嵌套
一个主机组还可以属于另外一个主机组。
示例：
```ini
[eastdc]
web1.example.com
db1.example.com
[westdc]
web2.example.com
db2.example.com
[dc:children]
eastdc
westdc
```
验证：
```bash
[ggg@contorller ~ 15:43:04]$ ansible --list-hosts -i inventory.example dc
  hosts (4):
    web1.example.com
    db1.example.com
    web2.example.com
    db2.example.com
```
子组中的主机组必须定义，否则会出现语法上的报错。
示例：
```ini
[eastdc]
web1.example.com
db1.example.com
[westdc]
web2.example.com
db2.example.com
#添加一个node1
[dc:children]
eastdc
westdc
node1
```
验证：
```bash
[ggg@contorller ~ 15:43:13]$ ansible --list-hosts -i inventory dc
 [WARNING]:  * Failed to parse /home/ggg/inventory.example with yaml plugin: Syntax
Error while loading YAML.   did not find expected <document start>  The error
appears to be in '/home/ggg/inventory.example': line 2, column 1, but may be elsewhere
in the file depending on the exact syntax problem.  The offending line appears to
be:  [eastdc] web1.example.com ^ here
 [WARNING]:  * Failed to parse /home/ggg/inventory.example with ini plugin:
/home/ggg/inventory.example:12: Section [dc:children] includes undefined group:
node1
 [WARNING]: Unable to parse /home/ggg/inventory.example as an inventory source
 [WARNING]: No inventory was parsed, only implicit localhost is available
 [WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'
  hosts (4):
    web1.example.com
    db1.example.com
    web2.example.com
    db2.example.com
```
### 动态主机清单
使用外部数据提供的信息动态生成Ansible清单信息。
### ansible-inventory 命令
通过不同的格式查看清单文件。
```bash
[ggg@contorller ~ 15:30:06]$ ansible-inventory --help
Usage: ansible-inventory [options] [host|group]
Options:
  --ask-vault-pass      ask for vault password
  --export              When doing an --list, represent in a way that is
                        optimized for export,not as an accurate representation
                        of how Ansible has processed it
  -h, --help            show this help message and exit
  -i INVENTORY, --inventory=INVENTORY, --inventory-file=INVENTORY
                        specify inventory host path or comma separated host
                        list. --inventory-file is deprecated
  --output=OUTPUT_FILE  When doing an --list, send the inventory to a file
                        instead of of to screen
  --playbook-dir=BASEDIR
                        Since this tool does not use playbooks, use this as a
                        substitute playbook directory.This sets the relative
                        path for many features including roles/ group_vars/
                        etc.
  --toml                Use TOML format instead of default JSON, ignored for
                        --graph
  --vars                Add vars to graph display, ignored unless used with
                        --graph
  --vault-id=VAULT_IDS  the vault identity to use
  --vault-password-file=VAULT_PASSWORD_FILES
                        vault password file
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number, config file location,
                        configured module search path, module location,
                        executable location and exit
  -y, --yaml            Use YAML format instead of default JSON, ignored for
                        --graph
  Actions:
    One of following must be used on invocation, ONLY ONE!
    --list              Output all hosts info, works as inventory script
    --host=HOST         Output specific host info, works as inventory script
    --graph             create inventory graph, if supplying pattern it must
                        be a valid group name
Show Ansible inventory information, by default it uses the inventory script
JSON format
```
示例清单：
```bash
[ggg@contorller ~ 15:30:06]$ vim inventory.example
```
```ini
app1.example.com
[webservers]
web1.example.com
web2.example.com
192.168.3.7
[dbservers]
db1.example.com
db2.example.com
192.0.2.42
[eastdc]
web1.example.com
db1.example.com
[westdc]
web2.example.com
db2.example.com
[dc:children]
eastdc
westdc
```
验证：
```bash
# 树形结构显示
[ggg@contorller ~ 15:31:16]$ ansible-inventory -i inventory.example --graph
@all:
  |--@dbservers:
  |  |--192.0.2.42
  |  |--db1.example.com
  |  |--db2.example.com
  |--@dc:
  |  |--@eastdc:
  |  |  |--db1.example.com
  |  |  |--web1.example.com
  |  |--@westdc:
  |  |  |--db2.example.com
  |  |  |--web2.example.com
  |--@ungrouped:
  |  |--app1.example.com
  |--@webservers:
  |  |--192.168.3.7
  |  |--web1.example.com
  |  |--web2.example.com
# yaml格式显示
[ggg@contorller ~ 15:32:10]$ ansible-inventory -i inventory.example --list -y
all:
  children:
    dbservers:
      hosts:
        192.0.2.42: {}
        db1.example.com: {}
        db2.example.com: {}
    dc:
      children:
        eastdc:
          hosts:
            db1.example.com: {}
            web1.example.com: {}
        westdc:
          hosts:
            db2.example.com: {}
            web2.example.com: {}
    ungrouped:
      hosts:
        app1.example.com: {}
    webservers:
      hosts:
        192.168.3.7: {}
        web1.example.com: {}
        web2.example.com: {}
```
### 用普通用户创建用户失败
```bash
[ggg@contorller ~ 17:22:39]$ cat inventory 
[controllers]
controller
[webservers]
node1
node3
[dbservers]
node2
node4
[ggg@contorller ~ 17:25:43]$ ansible -i inventory -m command -a 'cat /etc/hostname' -o all
node4 | CHANGED | rc=0 | (stdout) node4.ggg.cloud
node2 | CHANGED | rc=0 | (stdout) node2.ggg.cloud
node3 | CHANGED | rc=0 | (stdout) node3.ggg.cloud
node1 | CHANGED | rc=0 | (stdout) node1.ggg.cloud
controller | CHANGED | rc=0 | (stdout)
# 普通用户执行创建新用户：失败
[ggg@contorller ~ 17:26:02]$ ansible -i inventory -m command -a 'useradd zhangsan' -o all
node1 | FAILED! => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "cmd": "useradd zhangsan", "msg": "[Errno 2] 没有那个文件或目录", "rc": 2}
node4 | FAILED! => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "cmd": "useradd zhangsan", "msg": "[Errno 2] 没有那个文件或目录", "rc": 2}
node2 | FAILED! => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "cmd": "useradd zhangsan", "msg": "[Errno 2] 没有那个文件或目录", "rc": 2}
node3 | FAILED! => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "cmd": "useradd zhangsan", "msg": "[Errno 2] 没有那个文件或目录", "rc": 2}
controller | FAILED! => {"ansible_facts": {"discovered_interpreter_python": "/usr/bin/python"}, "changed": false, "cmd": "useradd zhangsan", "msg": "[Errno 2] 没有那个文件或目录", "rc": 2}
# 管理员用户执行创建新用户：成功
[ggg@contorller ~ 17:28:43]$ ansible -i inventory -m command -a 'useradd zhangsan' -u root -o all
node2 | CHANGED | rc=0 | (stdout) 
node4 | CHANGED | rc=0 | (stdout) 
node1 | CHANGED | rc=0 | (stdout) 
node3 | CHANGED | rc=0 | (stdout) 
controller | CHANGED | rc=0 | (stdout) 
#查看
[ggg@contorller ~ 17:28:51]$ ansible -i inventory -m command -a 'id zhangsan' -u root -o all
node4 | CHANGED | rc=0 | (stdout) uid=1001(zhangsan) gid=1001(zhangsan) 组=1001(zhangsan)
node1 | CHANGED | rc=0 | (stdout) uid=1001(zhangsan) gid=1001(zhangsan) 组=1001(zhangsan)
node3 | CHANGED | rc=0 | (stdout) uid=1001(zhangsan) gid=1001(zhangsan) 组=1001(zhangsan)
node2 | CHANGED | rc=0 | (stdout) uid=1001(zhangsan) gid=1001(zhangsan) 组=1001(zhangsan)
controller | CHANGED | rc=0 | (stdout) uid=1001(zhangsan) gid=1001(zhangsan) 组=1001(zhangsan)
```
## 管理 ANSIBLE 配置文件
### 配置文件位置和优先级
1. 环境变量 ANSIBLE_CONFIG
2. **./ansible.cfg**，当前位置中的 ansible.cfg，当前位置一般是项目目录。
3. ~/.ansible.cfg
4. /etc/ansible/ansible.cfg
**从上到下，优先级越来越低。**
**==建议==**：在当前目录下定义ansible.cfg文件。
**验证优先级**：
```bash
#默认是/etc/ansible/ansible.cfg
[ggg@contorller ~ 14:56:18]$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
......
#在家目录下创建配置文件
[ggg@contorller ~ 15:25:33]$ touch ~/.ansible.cfg
[ggg@contorller ~ 15:25:58]$ ansible --version
ansible 2.9.27
  config file = /home/ggg/.ansible.cfg
......
#在当前位置中的创建 ansible.cfg
[ggg@contorller ~ 15:24:21]$  mkdir web && cd web
[ggg@contorller web 15:28:17]$ ansible --version
ansible 2.9.27
  config file = /home/ggg/web/ansible.cfg
......
#修改环境变量 ANSIBLE_CONFIG
[ggg@contorller ~ 15:29:05]$ export ANSIBLE_CONFIG=/opt/ansible.cfg
[ggg@contorller ~ 15:29:54]$ sudo touch /opt/ansible.cfg
[ggg@contorller ~ 15:30:01]$ ansible --version
ansible 2.9.27
  config file = /opt/ansible.cfg
......
```
### 配置文件解析
ansible 默认配置文件 /etc/ansible/ansible.cfg。
Ansible 配置文件包括以下部分：
```bash
[ggg@contorller web 16:03:41]$ grep "^\[" /etc/ansible/ansible.cfg
[defaults]
[inventory]
[privilege_escalation]
[paramiko_connection]
[ssh_connection]
[persistent_connection]
[accelerate]
[selinux]
[colors]
[diff]
```
常用参数解析如下：
```ini
[defaults]
# inventory 指定清单文件路径
inventory = /etc/ansible/hosts
# 并发执行同一个任务的主机数量
forks          = 5
# ansible检查任务是否执行完成的时间间隔
poll_interval  = 15
# 连接登录到受管主机时是否提示输入密码
ask_pass = True
# 控制facts如何收集
# smart - 如果facts已经收集过了，就不收集了。
# implicit - facts收集，剧本中使用gather_facts: False关闭facts收集。
# explicit - facts不收集，剧本中使用gather_facts: True关闭facts收集。
gathering = implicit
# 收集facts范围
# all - gather all subsets
# network - gather min and network facts
# hardware - gather hardware facts (longest facts to retrieve)
# virtual - gather min and virtual facts
# facter - import facts from facter
# ohai - import facts from ohai
# You can combine them using comma (ex: network,virtual)
# You can negate them using ! (ex: !hardware,!facter,!ohai)
# A minimal set of facts is always gathered.
gather_subset = all
# 收集facts超时时间
gather_timeout = 10
# 变量注入，通过ansible_facts引用
inject_facts_as_vars = True
# 定义角色路径，以冒号分隔
roles_path = /etc/ansible/roles
# SSH是否检验 host key
host_key_checking = False
# 连接登录到受管主机时使用的用户身份
remote_user = root
# ansible 命令和ansible-playbook 命令输出内容存放位置
log_path = /var/log/ansible.log
# ansible 命令默认模块
module_name = command
# ssh 私钥文件位置
private_key_file = /path/to/file
# 默认ansible-vault命令的密码文件
vault_password_file = /path/to/vault_password_file
# 定义ansible_managed变量值
ansible_managed = Ansible managed
# 剧本执行过程中，遇到未定义的变量不报错
error_on_undefined_vars = False
# 系统告警启用
system_warnings = True
# 下架告警启用
deprecation_warnings = True
# 使用command和shell模块时，是否提示告警
command_warnings = False
# facts保存在哪里，例如redis
fact_caching = memory
[inventory]
# 启用的清单插件, 默认为: 'host_list', 'script', 'auto', 'yaml', 'ini', 'toml'
#enable_plugins = host_list, virtualbox, yaml, constructed
# 当清单源是一个目录的时候，忽略这些后缀的清单文件
#ignore_extensions = .pyc, .pyo, .swp, .bak, ~, .rpm, .md, .txt, ~, .orig, .ini, .cfg, .retry
[privilege_escalation]
# 连接到受管主机后是否需要进行权限提升或切换用户
become=True
# 使用何种方式进行用户切换或提权
become_method=sudo
# 用户切换或提权后的对应用户
become_user=root
# 进行用户切换或提权时是否提示输入密码
become_ask_pass=False
```
**说明**："#" 和 ";"开头的行，作为注释。
### 配置文件示例
对于基本操作， 使用 **[defaults]** 和 **[privilege_escalation]** 即可
```bash
#因为使用的是当前目录下配置文件
[ggg@contorller web 16:05:43]$ ansible --version
ansible 2.9.27
  config file = /home/ggg/web/ansible.cfg
......
[ggg@contorller web 16:04:44]$ ls
ansible.cfg  inventory
#当前目录下inventory 清单
[ggg@contorller web 16:02:50]$ cat inventory 
[controllers]
controller
[webservers]
node1
node3
[dbservers]
node2
node4
#当前目录下配置文件
[ggg@contorller web 16:03:18]$ cat ansible.cfg 
[defaults]
#主机清单采用当前目录的inventory
inventory      = ./inventory
#指定 ansible 远程 SSH 登录目标机器的默认用户名是ggg
remote_user = ggg
#提权自动配置
[privilege_escalation]
#开启自动提权
become=True
#提权方式选用sudo命令
become_method=sudo
#提权目标用户为 root
become_user=root
#sudo 免密
become_ask_pass=False
```
最终效果：
```bash
[ggg@contorller web 16:03:30]$ ansible webservers -a id
node1 | CHANGED | rc=0 >>
uid=0(root) gid=0(root) 组=0(root)
node3 | CHANGED | rc=0 >>
uid=0(root) gid=0(root) 组=0(root)
```
### ansible-config 命令
用于分析ansible命令的配置。
```bash
[ggg@contorller web 17:00:33]$ ansible-config -h
usage: ansible-config [-h] [--version] [-v] {list,dump,view} ...
View ansible configuration.
positional arguments:
  {list,dump,view}
    list            Print all config options
    dump            Dump configuration
    view            View configuration file
optional arguments:
  --version         show program's version number, config file location,
                    configured module search path, module location, executable
                    location and exit
  -h, --help        show this help message and exit
  -v, --verbose     verbose mode (-vvv for more, -vvvv to enable connection
                    debugging)
```
####  ansible-config view
查看当前ansible配合文件内容。
```bash
[ggg@contorller web 16:17:52]$ ansible --version|grep file
  config file = /home/ggg/web/ansible.cfg
[ggg@contorller web 16:17:22]$ ansible-config view
[defaults]
inventory      = ./inventory
remote_user = ggg
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
```
####  ansible-config dump
当前ansible生效的所有配置，包括所有默认值。
```bash
[ggg@contorller web 16:19:05]$ ansible-config dump
ACTION_WARNINGS(default) = True
AGNOSTIC_BECOME_PROMPT(default) = True
ALLOW_WORLD_READABLE_TMPFILES(default) = False
......
DEFAULT_HOST_LIST(/home/ggg/web/ansible.cfg) = [u'/home/ggg/web/inventory']
......
HOST_KEY_CHECKING(default) = True
......
```
####  ansible-config list
查看所有配置参数用途，配置位置等。
```bash
[ggg@contorller web 16:19:42]$ ansible-config list
.....
DEFAULT_HOST_LIST:
  default: /etc/ansible/hosts
  description: Comma separated list of Ansible inventory sources
  env:
  - {name: ANSIBLE_INVENTORY}
  expand_relative_paths: true
  ini:
  - {key: inventory, section: defaults}
  name: Inventory Source
  type: pathlist
  yaml: {key: defaults.inventory}
.....
```
### localhost 连接
默认Ansible连接到受管主机的协议为 smart （通常采用最有效的方式 - SSH）。如本地清单中并未指定localhost，Ansible会隐式设置localhost，并使用local连接类型连接localhost。
- local连接类型会**忽略remote_user的设置**，并且直接在本地系统上运行命令。
- 如果使用了特权提升，此时ansible将会在运行sudo时使用运行Ansible命令的账户的身份进行提权，而非remote_user所指定的账户。
**更改 localhost 连接方式**：清单中包涵 localhost。
## 运行 AD HOC 命令
### 实验环境
```bash
[ggg@contorller web 16:35:10]$ ansible-config view
[defaults]
inventory      = ./inventory
remote_user = ggg
[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
[ggg@contorller web 16:47:51]$ cat inventory 
[controllers]
controller
[webservers]
node1
node3
[dbservers]
node2
node4
```
### ansible AD HOC 命令
**命令作用**：
- **快速执行单个Ansible任务**，而不需要将它保存下来供以后再次运行。它们是简单的在线操作，无需编写playbook即可运行。
- **快速测试和更改很有用。**例如，您可以使用临时命令确保一组服务器上的/ etc/hosts文件中存在某一特定的行。您可以使用另一个临时命令在许多不同的计算机上高效重启一项服务，或者确保特定的软件包为最新版本。
**命令语法**：
```bash
ansible host-pattern -m module [-a 'module arguments'] [-i inventory]
```
- **host-pattern**，是inventory中定义的主机或主机组，可以为ip、hostname、inventory中的group组名、具有“,”或“*”或“:”等特殊字符的匹配型字符串，是必选项。
- **-m module**，module是一个小程序，用于实现具体任务。
- **-a 'module arguments'**，是模块的参数。
- **-i inventory**，指定inventory文件。
**命令执行结果颜色说明**：
Ansible的返回结果都非常友好，用3种颜色来表示执行结果： 
- <font color="#dd0000">红色：</font>表示执行过程有**异常**，一般会中止剩余所有的任务。
- <font color="#00dd00">绿色</font>：表示目标主机**已经是预期状态**，不需要更改 。
- <font color="#dddd00">黄色</font>：表示命令执行结束后目标有状态变化，并设置为预期状态，所有任务均正常执行。
### Ansible 部分模块
> Ansible 模块存放位置：/usr/lib/python*/site-packages/ansible
>
> 官网：[模块清单](https://docs.ansible.com/ansible/latest/collections/index_module.html)。
- **文件模块**
  - **copy**: 将控制主机上的文件复制到受管节点，类似于**scp**
  - **file**: 设置文件的权限和其他属性
  - **lineinfile**: 确保特定行是否在文件中
  - **synchronize**: 使用 **rsync** 将控制主机上的文件同步到受管节点
- **软件包模块**
  - **package**: 自动检测操作系统软件包管理器
  - **yum**: 使用 YUM 软件包管理器管理软件包
  - **apt**: 使用 APT 软件包管理器管理软件包
  - **gem**: 管理 Rubygem
  - **pip**: 从 PyPI 管理 Python 软件包
- **系统模块**
  - **firewalld**: 使用firewalld管理任意端口和服务
  - **reboot**: 重新启动计算机
  - **service**: 管理服务
  - **user、group**: 管理用户和组帐户
- **NetTools模块**
  - **get_url**: 通过HTTP、HTTPS或FTP下载文件
  - **nmcli**: 管理网络
  - **uri**: 与 Web 服务交互
### ansible-doc 命令
```bash
[ggg@contorller web 16:48:06]$ ansible-doc -h
Usage: ansible-doc [-l|-F|-s] [options] [-t <plugin type> ] [plugin]
plugin documentation tool
Options:
  -h, --help            show this help message and exit
  -j, --json            **For internal testing only** Dump json metadata for
                        all plugins.
  -l, --list            List available plugins
  -F, --list_files      Show plugin names and their source files without
                        summaries (implies --list)
  -M MODULE_PATH, --module-path=MODULE_PATH
                        prepend colon-separated path(s) to module library (default=~/.ansible/plugins/modules:/usr/share/ansible/plugins/modules)
  -s, --snippet         Show playbook snippet for specified plugin(s)
  -t TYPE, --type=TYPE  Choose which plugin type (defaults to "module").
                        Available plugin types are : ('become', 'cache',
                        'callback', 'cliconf', 'connection', 'httpapi',
                        'inventory', 'lookup', 'shell', 'module', 'strategy','vars')
  -v, --verbose         verbose mode (-vvv for more, -vvvv to enable
                        connection debugging)
  --version             show program's version number, config file location,
                        configured module search path, module location,
                        executable location and exit
See man pages for Ansible CLI options or website for tutorials
https://docs.ansible.com
```
示例：
```bash
# 查看模块清单及说明
[ggg@contorller web 16:48:56]$ ansible-doc -l
fortios_router_community_list                Configure community lists i...
azure_rm_devtestlab_info                     Get Azure DevTest Lab facts
......
# 查看模块清单及位置
[ggg@contorller web 16:48:56]$ ansible-doc -F
fortios_router_community_list    /usr/lib/python2.7/site-packages/ansibl....
azure_rm_devtestlab_info         /usr/lib/python2.7/site-packages/ansibl....
......
# 查看特定模块说明文档
[ggg@contorller web 16:48:56]$ ansible-doc user
> USER    (/usr/lib/python3.6/site-packages/ansible/modules/system/user.py)
    Manage user accounts and user attributes. For Windows targets, use the
[win_user] module instead.
  * This module is maintained by The Ansible Core Team
......
```
### command 模块
command 模块允许管理员在受管节点的命令行中运行任意命令。要运行的命令通过-a选项指定为该模块的参数。
```bash
[ggg@contorller web 16:51:20]$ ansible node1 -m command -a 'hostname'
node1 | CHANGED | rc=0 >>
node1.ggg.cloud
#-o合并
[ggg@contorller web 16:53:07]$ ansible node1 -m command -a 'hostname' -o
node1 | CHANGED | rc=0 | (stdout) node1.ggg.cloud
```
- command 模块执行的远程命令不受受管节点上的shell处理，无法访问shell环境变量，也不能执行重定向和传送等shell操作。
- 如果临时命令没有指定模块，Ansible默认使用command模块。
### shell 模块
shell模块允许您将要执行的命令作为参数传递给该模块。 Ansible随后对受管节点远程执行该命令。与command模块不同的是， 这些命令将通过受管节点上的shell进行处理。因此，可以访问shell环境变量，也可使用重定向和管道等shell操作。
```bash
#直接敲set:列出当前 Shell 所有本地变量、环境变量、shell 内置函数
[ggg@contorller web 16:53:22]$ ansible node1 -m command -a set
node1 | FAILED | rc=2 >>
[Errno 2] 没有那个文件或目录
[ggg@contorller web 16:56:11]$ ansible node1 -m shell -a set
node1 | CHANGED | rc=0 >>
BASH=/bin/sh
BASHOPTS=cmdhist:extquote:force_fignore:hostcomplete:interactive_comments:progcomp:promptvars:sourcepath
BASH_ALIASES=()
BASH_ARGC=()
BASH_ARGV=()
BASH_CMDS=()
......
```
### raw 模块
raw 模块，可以直接在远端主机shell中执行命令，**远端主机不需要安装Python**（特别是**针对网络设备**）。在大部分场景中，不推荐使用command、shell、raw模块执行命令，因为这些模块不具有幂等性。
```bash
[ggg@contorller web 16:59:45]$ ansible node1 -m raw -a 'echo "hello ansible" > /tmp/hello.txt'
node1 | CHANGED | rc=0 >>
Shared connection to node1 closed.
# 此处多了一个现实：断开连接，相当于通过ssh连接到受管节点执行命令。
[ggg@contorller web 17:00:04]$  ansible node1 -a 'cat /tmp/hello.txt'
node1 | CHANGED | rc=0 >>
hello ansible
# 对比shell模块
[ggg@contorller web 17:00:15]$ ansible node1 -m shell -a 'echo "hello ansible" > /tmp/hello.txt'
node1 | CHANGED | rc=0 >>
```
### ansible AD HOC 命令选项
临时命令选项优先级高于配置文件中配置。
| **配置文件指令** | **命令行选项**        |
| ---------------- | --------------------- |
| inventory        | -i                    |
| remote_user      | -u                    |
| ask_pass         | -k, --ask-pass        |
| become           | --become, -b          |
| become_method    | --become_method       |
| become_user      | --become-user         |
| become_ask_pass  | --ask-become-pass, -K |