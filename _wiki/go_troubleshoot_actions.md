---
layout: wiki
title: Go 故障排查操作指南
categories: Go
description: 
keywords: go
---

# 工具指令
## pprof
开启网页总览 http://127.0.0.1:10000/debug/pprof

进入交互
```
//go tool pprof http://127.0.0.1:10000/debug/pprof/heap //查看堆的使用，即内存使用情况
//go tool pprof http://127.0.0.1:10000/debug/pprof/profile //查看cpu耗时，会详细列出每个函数的耗时
//go tool pprof http://127.0.0.1:10000/debug/pprof/goroutine //当前在运行的goroutine情况以及总数
 -//还有 block/ mutex
```
交互后命 `top / list/ web 令`

参数 `-inuse_space -alloc_space`

## profile参数
```
flat flat% sum% cum cum%
5.95s 27.56% 27.56% 5.95s 27.56% runtime.usleep
4.97s 23.02% 50.58% 5.08s 23.53% sync.(*RWMutex).RLock
4.46s 20.66% 71.24% 4.46s 20.66% sync.(*RWMutex).RUnlock
```
* flat: 采样时，该函数运行时间，不包含等待子函数的时间
* flat%: flat / 总采样时间值
* sum%: 前面所有行的 flat% 的累加值
* cum: 采样时，该函数出现在调用堆栈的采样时间，包括函数等待子函数返回。因此 flat <= cum
* cum%: cum / 总采样时间值

## trace
下载20秒的trace记录

`遇到棘手问题时，查看trace会比较容易定位`
```
curl http://100.97.1.35:10078/debug/pprof/trace?seconds=20 > trace.out
go tool trace trace.out 查看
```
[线上 trace 传送门](https://mp.weixin.qq.com/s/Z9DoVGwdAtpbjealQLEMkw)



