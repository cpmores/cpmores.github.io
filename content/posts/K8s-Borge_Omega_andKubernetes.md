+++
title = 'K8s-Borge,Omega,and Kubernetes'
date = 2025-11-30T17:29:31+08:00
draft = true

+++

https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44843.pdf

K8s 的来时路，系统性的理解。

 # 三个容器编排系统的诞生

+ Borge 被用于处理 `long-running services` 和 `batch jobs` ，特别受 `Global Work Queue` 影响。Google 为此为 Linux 提供了大量容器支持;
+ Omega 是为了提高 Borg 的软件工程能力而被提出的。使用一个中心的事务系统保存状态，可以被各个集群访问;
+ Kubernetes 同样有核心持久化存储，但是 Omega 是直接暴露，K8s 是使用 REST API 来访问。对于程序员来说，编排更加容易。

> 现代 container 同时代表了隔离环境和镜像

## Application-Oriented Infrastructure

