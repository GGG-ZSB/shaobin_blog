---
title: "申请免费的https证书"
date: 2026-06-18
categories: [""]
---

Let's Encrypt 官方推荐使用 ACME 客户端获取证书，其中 Certbot 是最常用的工具，适配 Linux、Windows 等主流系统。
## CentOS 7 系统
### 1.安装 Certbot
```bash
[root@nginx ~ 16:00:05]# yum install certbot -y
```
**提示**：certbot 依赖 epel仓库。
### 2.发起证书申请
执行以下命令启动手动 DNS 验证模式
```bash
[root@nginx ~ 16:46:25]#  certbot certonly --manual --preferred-challenges dns -d www.shaobin.cloud
```
若需申请泛域名证书（如`*.ggg.cloud`），可将域名参数改为`-d *.ggg.cloud -d ggg.cloud`。
```bash
[root@nginx ~ 16:46:25]# certbot certonly --manual --preferred-challenges dns -d shaobin.cloud -d *.shaobin.cloud
```
### 3.完成 DNS 验证
```bash
[root@nginx ~ 16:46:25]#  certbot certonly --manual --preferred-challenges dns -d www.shaobin.cloud
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
# 输入邮箱地址
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): mage16196@163.com
Starting new HTTPS connection (1): acme-v02.api.letsencrypt.org
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.6-August-18-2025.pdf. You must agree
in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
# 接受服务协议
(Y)es/(N)o: Y
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
# 同意邮箱接收该组织发来各种信息
(Y)es/(N)o: Y
Account registered.
Requesting a certificate for laoma.cloud and *.laoma.cloud
Performing the following challenges:
dns-01 challenge for laoma.cloud
dns-01 challenge for laoma.cloud
# 根据提示添加第一条 TXT 记录
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.www.shaobin.cloud with the following value:
_tV59lCh9w02iMCsxQUoJDaBei4go9x3-RWgJrjBerY
Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
# 这里不要按回车，等 TXT 记录配置完成后再按回车
# 这里不要按回车，等 TXT 记录配置完成后再按回车
# 这里不要按回车，等 TXT 记录配置完成后再按回车
```
登录域名服务商（如阿里云、腾讯云）的 DNS 控制台，添加对应的 TXT 记录。以下截图是阿里云控制台。
![image-20260604191242573](../../../typora-user-images/image-20260604191242573.png)
==添加后需要等待一段时间==，然后通过以下命令验证记录是否生效，==直到==能查到该记录，再按==回车==继续。
```bash
[root@nginx ~ 18:24:42]# nslookup -type=TXT _acme-challenge.www.shaobin.cloud 
Server:		223.5.5.5
Address:	223.5.5.5#53
Non-authoritative answer:
_acme-challenge.www.shaobin.cloud	text = "_tV59lCh9w02iMCsxQUoJDaBei4go9x3-RWgJrjBerY"
Authoritative answers can be found from:
```
### 4.根据提示，再次添加一个TXT记录(若直接成功不同添加了)
```bash
Please deploy a DNS TXT record under the name
_acme-challenge.laoma.cloud with the following value:
F_wkFIItFwq0UUuIqSmFR2ZvonK8Hu5r-BdIm388-WE
Before continuing, verify the record is deployed.
(This must be set up in addition to the previous challenges; do not remove,
replace, or undo the previous challenge tasks yet. Note that you might be
asked to create multiple distinct TXT records with the same name. This is
permitted by DNS standards.)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
# 这里不要按回车，等 TXT 记录配置完成后再按回车
# 这里不要按回车，等 TXT 记录配置完成后再按回车
# 这里不要按回车，等 TXT 记录配置完成后再按回车
```
再次登录域名服务商（如阿里云、腾讯云）的 DNS 控制台，添加新的 TXT 记录。如上图
==添加后需要等待一段时间==，然后通过以下命令验证记录是否生效，直到能查到该记录，再按==回车==继续。
```bash
[root@nginx ~ 18:24:42]# nslookup -type=TXT _acme-challenge.www.shaobin.cloud 
.......
```
### 5.获取证书
```bash
Waiting for verification...
Resetting dropped connection: acme-v02.api.letsencrypt.org
Cleaning up challenges
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/www.shaobin.cloud/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/www.shaobin.cloud/privkey.pem
   Your certificate will expire on 2026-09-02. To obtain a new or
   tweaked version of this certificate in the future, simply run
   certbot again. To non-interactively renew *all* of your
   certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:
   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
验证通过后，证书会自动生成并存储在`/etc/letsencrypt/live/你的域名/`目录下，包含证书链文件`fullchain.pem`和私钥文件`privkey.pem`。
### 6.配置域名解析
![image-20260604191602241](../../../typora-user-images/image-20260604191602241.png)
```bash
```
当前未配置服务器，现用内网虚拟机ip地址解析
```bash
#查看证书存放位置
[root@nginx ~ 18:27:34]# ls /etc/letsencrypt/live/www.shaobin.cloud/
cert.pem  chain.pem  fullchain.pem  privkey.pem  README
```
| 文件名        | 作用                                          | Nginx 配置用途                              |
| ------------- | --------------------------------------------- | ------------------------------------------- |
| fullchain.pem | **完整证书链（域名证书 + 中间 CA 证书合并）** | `ssl_certificate 对应这个（nginx必填首选）` |
| privkey.pem   | **网站私钥（加密密钥，机密不能外泄）**        | `ssl_certificate_key 固定填这个`            |
| chain.pem     | 单独的中间根证书（分包拆分版，日常基本不用）  | 极少场景使用                                |
| cert.pem      | 仅单域名证书（不含中间证书）                  | 不用，优先 fullchain                        |
```bash
#配置 SSL/TLS
[root@nginx ~ 18:33:39]# vim /etc/nginx/conf.d/shaobin.conf
#将http重定向到https
server {
    listen 80;
    server_name www.shaobin.cloud;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name www.shaobin.cloud;
    ssl_certificate /etc/letsencrypt/live/www.shoabin.cloud/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/www.shaobin.cloud/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    root /usr/share/nginx/html;
    index index.html;
}
#配置欢迎界面
[root@nginx ~ 18:30:48]# vim /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>HTTPS 配置成功</title>
    <style>
        body { text-align: center; margin-top: 100px; font-size: 24px; color: #2c3e50; }
        .success { color: #27ae60; font-size: 36px; }
    </style>
</head>
<body>
    <div class="success">✅ HTTPS 配置成功！</div>
    <p>域名：www.shaobin.cloud</p>
    <p>SSL 证书已正常生效</p>
</body>
</html>
#重启nginx服务
[root@nginx ~ 18:39:59]# systemctl restart nginx
```
### 7.访问域名验证证书是否有效
![image-20260604192422969](../../../typora-user-images/image-20260604192422969.png)
验证成功，https未显示不安全
### 设置自动续期
Let's Encrypt 证书有效期为 90 天，可通过定时任务实现自动续期。例如 Linux 系统中，添加 crontab 定时任务：
```bash
# 每天凌晨2点检查证书，到期自动续期
echo "0 2 * * * /usr/bin/certbot renew --quiet" | tee -a /etc/crontab
```
### 证书续期后，网站如何更新证书
推荐解决方案：
1. 网站证书通过软连接指向生成的位置。（首选）
2. 网站证书直接指向生成的位置。
3. 配置监控脚本：证书发生变化后，复制到网站证书位置，并重启网站服务。