---
layout:     post
title:      "Linux下ASLR概述"
subtitle:   "记录Linux下内存随机化的情况，以Ubuntu为例。"
date:       2016-09-09 12:00:00
author:     "Readm"
header-img: "img/post-bg-2015.jpg"
tags:
    - 技术
    - Linux
    - 安全
---

## 动态库，堆栈和vsdo等。
ASLR在系统层面上分为3中情况：

+ `0` 不开启
+ `1` 保守开启(Conservative)，除了堆以外全部开启。
+ `2` 全部开启

该设置记录在`/proc/sys/kernel/randomize_va_space`中。

可以使用echo或者cat来更改和查看当前设置。

或者使用`sudo sysctl -w kernel.randomize_va_space=n`设置。

## 程序Image

在1和2中，程序本身编译关系到程序段是否随机化。即是否开启了PIE，gcc缺省并不开启。

*Note: linux在x86下开启pie会有明显的性能损耗，而在x64下则不会。*