---
title: 使用kubeadm搭建通用的Kubernetes集群
date: 2017-06-14 10:30:44
categories: Kubernetes
---

> `kubeadm`尚处于alpha阶段，安装中会卡住，目前尚未解决

## 安装`kubeadm`

### 前提条件

* 一台或多台运行`Ubuntu 16.04+`，`CentOS 7`或者`HypriotOS v1.0.1+`系统的机器。

* 每台机器拥有1GB及以上容量的内存（否则的话，留给你的应用空间就会很少了）

* 集群中的所有机器都能完全相连（公网互联还是私网互联都行）

### 安装`Docker`

在每一台机器上都要安装好Docker。推荐使用`v1.12`版本，但`v1.10`和`v1.11`也可以。`v1.13`以及`v17.03+`版本目前还没有被Kubernetes node小组测试与验证。想要获取安装指南，可以参看[Install Docker](https://docs.docker.com/engine/installation/)

### 安装`kubectl`

在每一台机器上，参看[install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)安装好kubectl。你只需要在master节点用到kubectl，但其它节点装上kubectl也可以使用。

### 安装`kubelet`和`kubeadm`

你将要在所有机器上安装下面这几个包：

* `kubelet`：这是Kubernetes最核心的组件。它在你的集群中每一台机器上运行，做一些比如启动`pods`和`containers`的工作。

* `kubeadm`：用来控制启动集群

备注：

## 参考文章

* [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
* [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
* [Creating a Custom Cluster from Scratch](https://kubernetes.io/docs/getting-started-guides/scratch/)