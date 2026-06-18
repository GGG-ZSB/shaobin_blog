---
title: "🚀 个人博客系统架构与部署说明（Hugo + Stack）"
date: 2026-06-18
tags: []
categories: []
---

## 📌 一、当前博客技术架构
当前个人博客基于以下技术栈构建：
### 1️⃣ 静态站生成器
- 工具：Hugo
- 版本：v0.162.0 (extended)
- 特点：
  - 基于 Go 语言
  - Markdown 驱动
  - 构建速度极快（毫秒级）
  - 适合技术博客与个人网站
------
### 2️⃣ 博客主题
- 主题名称：Hugo Theme Stack
- 仓库：
  - `github.com/CaiJimmy/hugo-theme-stack/v3`
特点：
- 卡片式 UI 设计
- 左侧个人信息栏
- 多语言支持（EN / ZH / JA）
- 分类 / 标签系统完善
- 适合个人技术博客
------
### 3️⃣ Web 服务器
- 服务：Nginx
- 系统：Ubuntu 24.04 LTS
作用：
- 托管 Hugo 生成的静态 HTML 文件
- 提供 HTTP / HTTPS 访问服务
网站目录：
```
/var/www/shaobin.cloud
```
------
### 4️⃣ 域名系统
- 域名：shaobin.cloud
- DNS 已解析至服务器公网 IP
访问方式：
```
https://shaobin.cloud
```
------
## 📌 二、博客运行机制
当前博客运行流程如下：
```
Markdown 文件（content/post）
        ↓
Hugo 构建
        ↓
生成 public/ 静态文件
        ↓
Nginx 读取 /var/www/shaobin.cloud
        ↓
浏览器访问 shaobin.cloud
```
------
## 📌 三、当前具备的功能
✔ Hugo 静态博客系统
✔ Stack 主题 UI
✔ 自定义头像与配置
✔ Nginx 部署成功
✔ 域名访问正常
✔ HTTPS（如已配置证书）
------
## 📌 四、当前存在的问题
### ❌ 1. GitHub 未完全接入
当前代码未完整推送到 GitHub 仓库，导致：
- 无法自动部署
- 无版本管理
- 无 CI/CD
------
### ❌ 2. 无在线写作后台
目前仍需：
```
SSH 登录服务器
→ 编辑 Markdown
→ hugo 构建
→ 手动部署
```
------
### ❌ 3. 无评论系统
尚未集成：
- GitHub 评论（Giscus）
------
## 📌 五、目标升级架构（最终形态）
升级后系统：
```
GitHub 仓库
      ↓
GitHub Actions 自动构建
      ↓
自动部署到服务器
      ↓
Nginx
      ↓
shaobin.cloud
```
并新增：
- /admin 在线写作后台（Decap CMS）
- GitHub 评论系统（Giscus）
- 全自动发布系统
------
## 📌 六、未来功能模块
### 1️⃣ 在线写博客（Decap CMS）
访问：
```
https://shaobin.cloud/admin
```
功能：
- 在线写 Markdown
- 图片上传
- 一键发布
- 自动同步 GitHub
------
### 2️⃣ 评论系统（Giscus）
基于 GitHub Discussions：
- GitHub 登录评论
- 无数据库
- 完全免费
- 与博客自动绑定
------
### 3️⃣ 自动部署（GitHub Actions）
实现流程：
```
提交文章
→ GitHub
→ 自动构建 Hugo
→ 自动更新服务器
```
------
## 📌 七、总结
当前博客已经完成：
✔ 80% 基础建设（Hugo + Nginx + 域名）
下一阶段目标：
🚀 GitHub 化
🚀 自动部署
🚀 在线写作
🚀 评论系统
最终将成为：
> 一个完全自动化的现代个人技术博客系统