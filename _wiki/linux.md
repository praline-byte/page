---
layout: wiki
title: Linux常用命令
categories: Linux
description: 类 Unix 系统下的一些常用命令和用法。
keywords: Linux
topmost: true
---

类 Unix 系统下的一些常用命令和用法。

## 磁盘相关
### 磁盘使用率
```shell script
df -lh
```
### 磁盘空间大小
```shell script
du -sh
```
### 磁盘IO
```shell script
sar -d -p 1
```

## 网络与端口
### 查看端口是否打开
```shell script
telnet 10.211.55.12 6379 
nc -v 10.211.55.12 6379
```

### 查看端口被什么进程占用
```shell script
sudo netstat -ltpn | grep :22
sudo lsof -n -P -i:22
```

### 查看进程监听的端口号
```shell script
sudo netstat -atpn | grep 1333
sudo lsof -n -P -p 1333 | grep TCP
```

## 压缩
```shell script
分卷压缩 zip -s SIZE origin.zip --out new.zip
解压 zip spiltfile.zip -s=0 --out single.zip
加密压缩 -P
```

## 实用命令汇总
### lsof
查看打开的文件列表。

> An  open  file  may  be  a  regular  file,  a directory, a block special file, a character special file, an executing text reference, a library, a stream or a network file (Internet socket, NFS file or UNIX domain socket.)  A specific file or all the files in a file system may be selected by path.

#### 查看端口被什么进程占用

```shell script
lsof -n -P -i:22
```

#### 查看进程监听的端口号

```shell script
sudo lsof -n -P -p 1333 | grep TCP
```

#### 查看某个文件被谁占用

```sh
lsof .linux.md.swp
```

#### 查看某个用户占用的文件信息

```sh
lsof -u mazhuang
```

`-u` 后面可以跟 uid 或 login name。
