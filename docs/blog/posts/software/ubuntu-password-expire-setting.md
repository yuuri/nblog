---
layout: post
date: 2025-12-18
tags: [linux,password,expire,ubuntu]
categories:
    - software
---

# ubuntu 密码过期重置密码

ubuntu 设置密码90天后自动过期，没有在过期前修改密码，可能会进入不去图形界面，可按如何操作进行设置

## 1.登录页面进入Recovery Mode
登录页面选择Recovery Mode 登录选项

## 2.重新挂载为可写（非常关键）
mount -o remount,rw /

## 3.修改过期用户的密码
passwd your_username
