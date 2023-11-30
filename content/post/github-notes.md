---
title: "Github Notes"
subtitle:    ""
description: ""
date: 2022-10-03
draft: false
author:      "Ashley Hsu"
image:       ""
tags:        ["GitHub"]
categories:  ["Tech" ]
---

## 配置 GitHub 金鑰
流程：
1. 產生 key pair
2. 在 GitHub 放 public key

### 產生 key pair
```bash
cd ~/.ssh
ssh-keygen -t ed25519 -C "your_email@example.com"
```
不設密碼的話直接按 enter
```
Enter file in which to save the key (/home/username/.ssh/id_ed25519): /home/username/.ssh/github_key
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
```

### 在 GitHub 放 public key
複製公鑰放到 GitHub
```
cat id_ed25519.pub
```
貼到這邊：
`Settings` -> `SSH and GPG keys` → `New SSH key`

### 連線測試
```
ssh -T git@github.com
```

### `~/.ssh/config`
```
 Host github.com
  HostName github.com
  User ashirleyshe
  IdentityFile ~/.ssh/id_rsa
```
## 多個帳號
### 查看 config
```
git config --list
```

### 改 config
```
git config user.name XXX
git config user.email XXX
```
### add remote
```
git remote add origin git@niugiao:niujiao-yum/niugiao-contracr.git
```