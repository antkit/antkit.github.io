---
title: Ubuntu 用户与权限
date: 2017-06-06 15:49:40
categories: Ubuntu
tags: Ubuntu
---

## 增加用户
可以使用 adduser 和 useradd 来给 Ubuntu 系统添加用户，下面以 adduser 为例。
``` bash
# 添加用户 username
sudo adduser username

# 将用户 username 添加到 sudo 组
sudo adduser username sudo
```

## 删除用户
删除用户需要用到 userdel，前提要求目标用户当时未登录。
``` bash
# 删除用户 username
sudo userdel username
```

## 修改密码
使用 passwd 修改密码。
``` bash
# 修改当前用户的密码
passwd

# 修改别人（username）的密码
sudo passwd username
```

## 文件系统权限
``` bash
# 只有所有者有读和写的权限
chmod 600 ×××
# 所有者有读和写的权限，组用户只有读的权限
chmod 644 ×××
# 只有所有者有读和写以及执行的权限
chmod 700 ×××
# 每个人都有读和写的权限
chmod 666 ×××
# 每个人都有读和写以及执行的权限
chmod 777 ×××
```