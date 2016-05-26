title: Deploy Openstack-Mitaka with Kolla in Ubuntu 14.04
date: 2016-05-26 22:53:00
tags: openstack
---
> 在Kolla之前部署过Openstack的都知道这是一件多么繁琐的任务，不过熟悉Chef或Puppe
的童鞋可能说完全可以通过这些工具自动化的部署，但是对于没接触过这些工具的人来说，使用它们来编写Openstack自动部署脚本同样令人头疼；正是在这样的大背景下，Kolla横空出世，其宗旨是: "to provide production-ready containers and deployment tools for operating OpenStack clouds".
使用Kolla,部署Openstack可以说是傻瓜式的操作体验，下面来介绍基于ubuntu 14.04, 使用Mitaka版本Kolla进行多节点Openstack部署的详细步骤，以及本人遇到的所有大坑...
<!--more-->

## 背景
由于条件限制，我是使用Vagrant(1.7.4)起了两个虚拟机分别作为控制节点和计算节点，虚拟机的基本配置见(链接). 不过在实际多节点部署时，出于高可用的目的，一般会有多个控制节点，并且会把网络服务Neutron和存储服务单独放在不同的节点上，在本文中，这些服务都仍在同一个控制节点上.

## 基本环境
1. 升级内核(all)
一般来讲ubuntu 14.04 默认的内核3.13是支持docker的各项特性的，但是一些老版本的内核可能跟Docker使用的AUFS和OverlayFS存在一些兼容问题，以防万一，升级内核到最新版本肯定没错.
```
$ sudo apt-get -y install linux-image-generic-lts-wily
```
安装完成之后，需要重启机器才能生效.
2. 安装Docker(all)
Kolla是将Openstack的所有服务组件都放在容器中，Docker当然是需要安装的。使用一下脚本安装最新的Docker:
```
$ sudo curl -sSL https://get.docker.io | bash
```
安装完成后,docker只能在root用户下使用，使用一下命令将其他用户添加到docker组中:
```
$ sudo usermode -aG docker $USERNAME
```
接着需要对docker-engine服务进行相关配置，
```
$ sudo mount --make-shared /run
```
重启docker,
```
$ sudo service docker restart
```
最后更新Docker的python库，
```
$ sudo pip install -U docker-py
```

3. 安装NTP(all)
Openstack中很多组件都有消息的传递，为保证系统的正确性，需要进行时钟的同步，推荐使用ntp进行时钟同步:
```
$ sudo apt-get install ntp
```

4. 安装Ansible(controller)
推荐安装1.9.4版本,使用pip进行安装:
```
sudo pip install -U ansible==1.9.4
```
