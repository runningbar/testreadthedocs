---
title: Ubuntu NFS使用
date: 2017-06-05 14:22:01
categories: 经验
---

> 在`Ubuntu16.04`下测试通过

## NFS安装

CentOS下查看NFS是否已经安装：

```
rpm -qa | grep "nfs"
```

Ubuntu下安装NFS服务器：

``` 
sudo apt install nfs-kernel-server
```

Ubuntu下安装NFS客户端：

```
sudo apt install nfs-common
```

## 在NFS服务器上指定共享目录

指定共享目录有2种方式，分别有不同的目的：一种适用于通用的文件存储；另一种则适用于带权限管理的文件存储。

root用户可以在系统的任何位置做任何事情。然而，NFS挂载的目录不是这个系统的一部分，所以默认情况下，NFS服务器会拒绝执行需要超级用户权限的操作。这个默认的限制意味着，客户端系统的root用户无法在NFS挂载目录下，以root角色写文件，重新指定文件拥有者，或者执行其它超级用户操作。

这里先介绍默认方式的共享目录配置方法，它适用于大多数场景，比如作为内容管理系统存储上传的文件，或者为用户开辟空间共享项目文件。

首先，创建一个待共享的目录：

```
mkdir -p /home/mine/nfs
```

**由于是默认的共享方式，NFS会将客户机上的所有`root`操作都映射为`nobody:nogroup`身份，因此，我们需要修改该目录的拥有权，以便匿名身份可以读写该目录**：

```
sudo chown nobody:nogroup /home/mine/nfs
```

> 在下文的`/etc/exports`配置中，我们可以使用`all_squash`选项，将客户机上的所有用户都映射为匿名用户，这样就保证了所有人都能够存取该目录。


接下来，配置NFS导出：

使用root权限打开`/etc/exports`文件：

```
sudo vim /etc/exports
```

这个文件有一段注释内容，展示了每一个配置行的基本结构，语法大致是这样的：

```
directory_to_share client(share_option1,...,share_optionN)
```

对于每一个要共享的目录都要配置一行：

```
/home/mine/nfs 192.168.100.180(rw,sync,all_squash,insecure)
```

这意味着将`/home/mine/nfs`目录共享给`192.168.100.180`机器使用，如果想共享给所有主机，也可以换成`*`：

```
/home/mine/nfs *(rw,sync,all_squash,insecure) 
/*option list之间不能有空格，否则会报错：bad option list*/
```

共享设置一览：

* rw： 允许客户端对卷有读写权限
* sync： 强制NFS在回复之前将改变写入磁盘。因为回复反映了远程卷的实际状态，所以这样可以产生更加稳定一致的环境。然而，它也降低了文件操作的速度。
* all_squash：客户机上的任何用户读写该共享目录时，都映射成匿名用户，这样就**便于所有用户对文件的存取**。
* insecure：允许客户机从大于等于`1024`的端口来访问NFS，如果不写明此选项，有时候客户机`mount`时会报错`access denied`。

Ubuntu下操作NFS服务器：

```
sudo systemctl start/stop/restart/status nfs-kernel-server.service
```

## 在客户端创建挂载点

为了在客户端机器上使用远程共享目录，我们需要将该目录挂载到本地一个空目录上。

> 如果待挂载目录下已经有文件或文件夹，那么一旦你挂载了NFS共享目录，原有的文件就会被隐藏掉。所以要确保你的挂载目录若已存在，那么它得是一个空目录。

创建挂载目录：

```
mkdir -p /home/client/nfs
```

挂载NFS共享目录：

```
sudo mount 192.168.1.100:/home/mine/nfs /home/client/nfs //192.168.1.100是NFS服务器IP
```

检查挂载情况：

```
df -h
```

卸载共享目录的挂载：

```
sudo umount /home/client/nfs
```

okay，现在无论在NFS服务器，还是客户机上，任何人都能对`nfs`共享目录进行读写操作了。


## 参考文章

* [NFS](https://help.ubuntu.com/lts/serverguide/network-file-system.html)
* [How To Set Up an NFS Mount on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-16-04)
* [exports(5) - Linux man page](https://linux.die.net/man/5/exports)