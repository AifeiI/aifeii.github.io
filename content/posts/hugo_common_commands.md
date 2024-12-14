+++
title = 'Hugo常用命令'
date = 2024-12-14T23:51:58+08:00
draft = true
+++

## 常用命令

```shell
# 创建新的文章
hugo new /path/file.md

# 启动本地预览服务端，不包括草稿内容
hugo server

# 启动本地预览服务端，并包括草稿内容
hugo server -D

# 将指定文章从草稿状态转为可发布状态
hugo undraft /path/file.md
```