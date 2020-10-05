---
layout: wiki
title: Java 故障排查操作指南
categories: Java
description: 
keywords: java
---

一般的，线上故障主要包括 CPU，GC，内存，磁盘和网络五个方面的问题

问题往往反映在多个部分，比如内存泄漏引起频繁 GC，问题需要定位到本质

这里用于记录**故障快速定位**，原因分析和应对方法[传送门]()(待补充)
## CPU
常用于负载过高`大于 逻辑核心数 * 0.7`，cpu 100%等情况 
### 堆栈四部曲
#### 1.拎出进程
```
top -c
```
或者 `ps | grep {keyword}` 
#### 2.抓出此进程下 cpu 使用率高的线程
```
top -H -p {ppid}
```
#### 3.线程十六进制转换
```
printf "%x\n" {pid}
```
#### 4.曝光堆栈信息
```
jstack pid |grep 'nid' -C5 –color
```

通常我们关注 BLOCKED，WAITING 和 TIMED_WAITING

也可以使用命令对堆栈有整体的把握
```
cat jstack.log | grep "java.lang.Thread.State" | sort -nr | uniq -c
```
### 上下文切换
```
pidstat -w {pid}
```
`cswch 和 nvcswch 表示自愿及非自愿切换`

## GC
### 观察 gc 分代变化
```
jstat -gc {pid} 1000
```
### 分析 gc 日志
```
启动参数中加上 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps 来开启GC日志
```
### FullGC
* 并发阶段失败：在并发标记阶段，MixGC之前老年代就被填满了，那么这时候G1就会放弃标记周期。这种情况，可能就需要增加堆大小，或者调整并发标记线程数-XX:ConcGCThreads。
* 晋升失败：在GC的时候没有足够的内存供存活/晋升对象使用，所以触发了Full GC。这时候可以通过-XX:G1ReservePercent来增加预留内存百分比，减少-XX:InitiatingHeapOccupancyPercent来提前启动标记，-XX:ConcGCThreads来增加标记线程数也是可以的。
* 大对象分配失败：大对象找不到合适的region空间进行分配，就会进行fullGC，这种情况下可以增大内存或者增大-XX:G1HeapRegionSize。
* 程序主动执行System.gc()

### 观察总线程数量
```
ls -l /proc/pid/task | wc -l
```

## 内存
> 以下信息将被挪走到 **原因分析和应对方法**

OOM 通常有三类
* 堆内存溢出，java.lang.OutOfMemoryError :Java heap space
>分析堆内存dump文件，确认是内存泄露/ 还是内存溢出
 如果是内存溢出，则需要考虑，是否有长生命周期的对象可缩短，是否可精简大对象
* 栈内存溢出
    * 如果栈深度大于虚拟机所允许的最大深度，则StackOverflowError
    * 如果扩展栈，无法得到内存空间，则OutOMemoryError
    * 如果使用虚拟机默认参数,栈深度在大多数情况下(因为每个方法压入栈的帧大小并不是一样的,所以只能说在大多数情况下)达到1000〜2000完全没有问题,对于正常的方法调用(包括递归),这个深度应该完全够用了
    * -Xss 为指定每个线程栈大小，如果在多线程下抛出溢出，可以考虑，减少堆/方法区的内存空间
      栈空间(线程数 * Xss) = 机器内存 - 堆内存 - 方法区内存
* 方法区/常量池 溢出 java.lang.OutOfMemoryError :PermGen space
    * 注意字符串的大小
    * 经常动态生成大量Class的应用中,需要特别注意类的回收状况。这类场景除了上面提到的程序使用了CGLib字节码增强和动态语言之外,常见的还有:大量JSP或动态产生JSP文件的应用(JSP第一次运行时需要编译为Java类 )、基于OSGi的应用(即使是同一个类文件,被不同的加载器加载也会视为不同的类)等

### 查看堆内存  
dump 内存文件并用 JMAP 定位
```
jmap -dump:format=b,file=filename {pid}
```
`可以在启动参数中指定 -XX:+HeapDumpOnOutOfMemoryError 来保存 OOM 时的 dump 文件`

## 磁盘
### 磁盘空间
```
df -hl
```
### 磁盘性能
```
iostatiostat -d -k -x
```
`rrqpm/s 以及 wrqm/s 分别表示读写速度`
### 进程读写情况
```
cat /proc/{pid}/io
```
`lsof -p {pid} 确定具体的文件读写情况`

## 网络
### TCP 队列溢出
```
netstat -s | egrep "listen|LISTEN"
```
### TIME_WAIT 和 CLOSE_WAIT 
是否短时间建立起大量链接又断开

查看 TIME_WAIT 和 CLOSE_WAIT 数量
```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}
```
`用ss命令会更快ss -ant | awk '{++S[$1]} END {for(a in S) print a, S[a]}`

--- 

说在最后
>解决问题的最佳手段就是不解决，在问题发生之前探测告警。
>将服务接入监控系统，配上合理的阈值和告警，将问题扼杀在摇篮里。
