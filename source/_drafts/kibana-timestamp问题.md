---
title: kibana timestamp问题
date: 2017-05-27 17:05:47
tags: 笔记
categories: ELK
---

##1.问题描述

nifi产生的日志通过nifi处理器写入到elasticsearch中，然后在kibana中查找，发现时间总是对不上，按照网上的方案改写了kibana的tz参数为UTC，时间对了，但是最近的15分钟等操作是不对的

##2.解决方案
指导思想：存储数据和显示数据分离
在写入elasticsearch之前对nifi采用logback产生的日志做一个jolt，为timestamp加一个 +08:00
