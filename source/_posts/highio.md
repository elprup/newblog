---
date: 2017-01-12T02:31:15Z
title: 记一次服务器宕机恢复过程
slug: highio_database_recover
---

## 服务器介绍

这是一台数据库服务器，运行 MySQL 和 Mongodb。用于平台用户系统，统计系统的数据存储。

## 宕机重启

收到网页无法访问报警后，打开网页发现超时，排查后发现 db 服务器已经无法 ssh 登录。只能选择关机重启。控制台选择关机后失败，只能强制关机。强制关机成功后能 ssh 登录。

<!-- more -->

## 网页恢复，访问缓慢

重启后数据库运行正常，网页打开正常。但半数机会需等待 20 多秒才显示。服务器高 Load，有很高的 iowait。怀疑底层磁盘问题。这台服务器使用了 500G 的云磁盘挂载。联系美团云后回复磁盘没问题。查看 mongodb 日志发现慢操作写入等待时间有 20 多秒，导致读操作被阻塞。

iowait 高问题成为事情的关键。

## 了解 iowait（wa）

iowait 表示在一个采样周期内有百分之几的时间属于以下情况：CPU 空闲、并且有仍未完成的 I/O 请求。IO 特指文件 IO，不包括网络 IO[SO 解释](http://serverfault.com/questions/37441/does-iowait-include-time-waiting-for-network-calls)。所以高 iowait 必然和磁盘有关。

## 查找高 io 进程

用 vmstat 命令看到的情况是，服务器在两种忙的状态下切换，一种是高 sy，一种是高 wa。于是用 iotop 查看实时写入速度。发现大部分时间都没有请求，只是突然有 mysql 和 mongo 的请求。

这时我以为 IO 写入不忙，但后来发现问题不是这样。用 vmstat 可以发现突发的请求 bo 数可以到达 13000 左右，换算成数据量是 42MB。这种 bo 出现往往导致 wa 持续高几十秒。其实 iotop 的实时统计并不准确，可能采集数据是 io 写发起的数据量，而大的 io 鞋发起后磁盘真实的写入速度没有准确显示，所以造成我认为磁盘 io 不忙的错觉。

然后用 iotop 的`a`键查看累积写入，发现 mysql 写入非常高。原来异常重启后的 mysql 会检查所有表，导致了特别高的 IO。于是重启 mysql，中断了检查操作，问题解决。

## 最后

内存，IO，CPU 是系统最重要的几大运维内容。这次事件一方面说明磁盘 IO 的排查首先应该查找高 IO 进程，另一方面也提醒了关键数据库要有主从备份的重要性。
