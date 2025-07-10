---
layout: post
title: 配置多个GitHub账号的SSH密钥管理
slug: rc002
date: 2025-01-10 15:30
status: publish
author: Xuxx
categories: 
  - 日常
tags: 
  - Git
  - SSH
  - GitHub
excerpt: 如何配置多个GitHub账号的SSH密钥，实现不同项目使用不同账号进行Git操作
---

## 背景

当你需要同时使用多个GitHub账号（如个人账号和工作账号）时，需要配置不同的SSH密钥来区分身份。本教程将详细介绍如何配置多个GitHub账号的SSH密钥管理。

## 1. 生成SSH密钥对

为每个GitHub账号生成独立的SSH密钥对：

```bash
# 生成个人账号的SSH密钥
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github_personal -C "your-personal-email@example.com"

# 生成工作账号的SSH密钥
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github_work -C "your-work-email@example.com"
```

## 2. 配置SSH Config文件

创建或编辑 `~/.ssh/config` 文件：

```bash
# 个人GitHub账号
Host github.com-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    PreferredAuthentications publickey
    IdentitiesOnly yes

# 工作GitHub账号
Host github.com-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    PreferredAuthentications publickey
    IdentitiesOnly yes

# 默认GitHub配置（可选）
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    PreferredAuthentications publickey
    IdentitiesOnly yes
```

设置文件权限：

```bash
chmod 600 ~/.ssh/config
```

## 3. 添加SSH公钥到GitHub

获取公钥内容：

```bash
# 个人账号公钥
cat ~/.ssh/id_ed25519_github_personal.pub

# 工作账号公钥
cat ~/.ssh/id_ed25519_github_work.pub
```

将对应的公钥分别添加到对应的GitHub账号：
1. 登录GitHub → Settings → SSH and GPG keys → New SSH key
2. 粘贴公钥内容并保存

## 4. 测试SSH连接

```bash
# 测试个人账号连接
ssh -T github.com-personal

# 测试工作账号连接
ssh -T github.com-work
```

成功时会显示：`Hi [username]! You've successfully authenticated, but GitHub does not provide shell access.`

## 5. 使用不同的SSH配置

### 克隆仓库时

```bash
# 个人项目
git clone git@github.com-personal:username/repo.git

# 工作项目
git clone git@github.com-work:username/repo.git
```

### 修改现有项目的remote URL

```bash
# 修改为个人配置
git remote set-url origin git@github.com-personal:username/repo.git

# 修改为工作配置
git remote set-url origin git@github.com-work:username/repo.git
```

## 6. 批量配置脚本

创建脚本自动配置多个项目：

```bash
#!/bin/bash

# 根据组织名称自动配置SSH密钥
projects=(
    "project1"
    "project2"
    "project3"
)

for project in "${projects[@]}"; do
    project_path="/path/to/projects/$project"
    if [ -d "$project_path" ]; then
        cd "$project_path"
        remotes=$(git remote)
        
        for remote in $remotes; do
            current_url=$(git remote get-url "$remote")
            
            # 根据仓库URL判断使用哪个配置
            if [[ "$current_url" == *"work-org"* ]]; then
                new_url=$(echo "$current_url" | sed 's|git@github.com:|git@github.com-work:|')
            else
                new_url=$(echo "$current_url" | sed 's|git@github.com:|git@github.com-personal:|')
            fi
            
            git remote set-url "$remote" "$new_url"
        done
    fi
done
```

## 可选配置：使用443端口

### 什么时候需要443端口？

在某些网络环境中（如公司网络），标准的SSH端口（22）可能被防火墙阻止。GitHub提供了通过443端口进行SSH连接的选项。

### 配置443端口SSH

修改 `~/.ssh/config` 文件：

```bash
# 个人GitHub账号（使用443端口）
Host github.com-personal
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519_github_personal
    PreferredAuthentications publickey
    IdentitiesOnly yes

# 工作GitHub账号（使用443端口）
Host github.com-work
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
    PreferredAuthentications publickey
    IdentitiesOnly yes
```

### 添加SSH主机密钥到known_hosts

**为什么需要修改known_hosts？**

当你改变SSH连接的主机名（从 `github.com` 到 `ssh.github.com`）或端口时，SSH客户端会遇到新的主机密钥。为了安全验证，需要将GitHub的SSH主机密钥添加到 `~/.ssh/known_hosts` 文件中。

```bash
# 添加GitHub的SSH主机密钥（443端口）
ssh-keyscan -t rsa -p 443 ssh.github.com >> ~/.ssh/known_hosts
```

这个命令的作用：
- `ssh-keyscan`：扫描SSH主机密钥
- `-t rsa`：指定密钥类型为RSA
- `-p 443`：指定端口为443
- `ssh.github.com`：GitHub的SSH服务器地址
- `>> ~/.ssh/known_hosts`：将密钥追加到known_hosts文件

### 测试443端口连接

```bash
ssh -T github.com-work
```

## 故障排除

### 1. 权限被拒绝

```bash
# 检查SSH代理中的密钥
ssh-add -l

# 如果没有密钥，添加到SSH代理
ssh-add ~/.ssh/id_ed25519_github_personal
ssh-add ~/.ssh/id_ed25519_github_work
```

### 2. 连接超时或被拒绝

- 检查网络连接
- 尝试使用HTTPS代替SSH
- 使用443端口配置

### 3. 仓库不存在错误

- 确认仓库URL正确
- 检查GitHub账号权限
- 验证SSH密钥是否正确添加到对应账号

## 总结

通过合理配置SSH密钥和config文件，可以优雅地管理多个GitHub账号，实现不同项目使用不同身份进行Git操作。记住定期检查和更新SSH密钥，确保账号安全。