---
layout: post
title: "WSL 不加载 windows PATH 以提高自动补全性能"
subtitle: ""
author: "KodiakAS"
header-img: "img/database-bg.png"
header-mask: 0.3
tags:
  - WSL
---

wsl加载win环境变量会导致zsh的自动补全延迟大幅提高

编辑`/etc/wsl.conf`

```ini
[interop]
enabled=true
appendWindowsPath=false
```

重启wsl生效

在bashrc或zshrc手动添加必要环境变量

```bash
# vscode
export PATH=/mnt/c/Users/KodiakAS/AppData/Local/Programs/Microsoft\ VS\ Code/bin:/usr/lib/wsl/lib:$PATH

# 用于vscode插件识别wsl，如不添加，`su -`重新加载环境变量后vscode插件无法正常工作
export WSL_DISTRO_NAME=Ubuntu-20.04
```