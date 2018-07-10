---
title: git 常见问题
date: 2018-07-09 21:52:05
tags: git
---

### 推送本地分支至远端分支
```
git init
```
初始化
```
git add
```
添加文件夹中所有文件
```
git commit -m "init git dir"
```
提交该添加记录
```
git remote add origin url
```
添加该远程库url至本地并起别名origin
```
$ git push origin <local branch name>:<remote branch to push into>
```
推送本地库至origin远程分支

