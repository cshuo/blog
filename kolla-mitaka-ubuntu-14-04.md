title: Deploy Openstack-Mitaka with Kolla on Ubuntu 14.04
date: 2016-05-26 22:53:00
category: OpenStack
tags: openstack
---
> 在Kolla之前部署过Openstack的都知道这是一件多么繁琐的任务，不过熟悉Chef或Puppet的人可能说完全可以通过这些工具自动化的部署，但是对于没接触过这些工具的人来说，使用它们来编写Openstack自动部署脚本同样令人头疼；正是在这样的大背景下，Kolla横空出世，其宗旨是:`"to provide production-ready containers and deployment tools for operating OpenStack clouds"`.使用Kolla,部署Openstack可以说是傻瓜式的操作体验，下面来介绍基于ubuntu 14.04, 使用Mitaka版本Kolla进行多节点Openstack部署的详细步骤.
<!--more-->

---

## 背景
由于条件限制，我是使用Vagrant(1.7.4)起了两个虚拟机分别作为控制节点和计算节点，点击[这里](https://github.com/cshuo/kolla/blob/stable/mitaka/Vagrantfile)查看虚拟机的基本配置. 不过在实际多节点部署时，出于高可用的目的，一般会有多个控制节点，并且会把网络服务Neutron和存储服务单独放在不同的节点上，在本文中，这些服务都放在同一个控制节点上.

## 基本环境
> all: 控制节点和计算节点都需配置;  controller: 控制节点;  compute: 计算节点

1. 升级内核(all)    
一般来讲ubuntu 14.04 默认的内核3.13是支持docker的各项特性的，但是一些老版本的内核可能跟Docker使用的AUFS和OverlayFS存在一些兼容问题，以防万一，升级内核到最新版本肯定没错.    
`$ sudo apt-get -y install linux-image-generic-lts-wily`   
安装完成之后，需要重启机器才能生效.
2. 修改hosts(all)
确保控制节点可以解析计算节点hostname,hosts文件只保留127.0.0.1以及ip的解析, controller的hosts实例如下：
```
127.0.0.1   localhost
20.0.2.11   controller
20.0.3.11   compute
```
3. 安装Docker(all)    
Kolla是将Openstack的所有服务组件都放在容器中，Docker当然是需要安装的。使用一下脚本安装最新的Docker:    
`$ sudo curl -sSL https://get.docker.io | bash`   
安装完成后,docker只能在root用户下使用，使用一下命令将其他用户添加到docker组中:  
`$ sudo usermode -aG docker $USERNAME`   
接着需要对docker-engine服务进行相关配置，    
`$ sudo mount --make-shared /run`    
重启docker,   
`$ sudo service docker restart`    
最后更新Docker的python库，    
`$ sudo pip install -U docker-py`    
4. 安装NTP(all)    
Openstack中很多组件都有消息的传递，为保证系统的正确性，需要进行时钟的同步，推荐使用ntp进行时钟同步:    
`$ sudo apt-get install ntp`
5. 安装Ansible(controller)    
推荐安装1.9.4版本,使用pip进行安装:
`$ sudo pip install -U ansible==1.9.4`
6. 安装ssh密钥    
Kolla用Ansible进行安装配置的部分操作需要root权限，为操作方便，我们选择配置控制节点可以用root用户ssh到计算节点.     
    - 计算节点允许root ssh(compute)    
    `$ vi /etc/ssh/sshd_config`    
    "PermitRootLogin Without-password" -> "PermitRootLogin yes"   
    `$ sudo service ssh restart`
    - 生成控制节点root用户密钥(controller)     
    `$ sudo su`    
     `# ssh-keygen`    
    - 拷贝密钥至所有节点(all)    
    `$ sudo echo "$KEYS" >> /root/.ssh/authorized_keys`
    - 验证无密码登陆    
    `# ssh root@compute`    
    `# ssh root@controller`    
7. 安装openstack客户端
在运行Openstack CLI的机器上安装Openstack的客户端，这里即控制节点.     
    - 安装依赖
    `$ sudo apt-get install -y python-dev libffi-dev libssl-dev gcc`
    - 安装客户端    
    `$ pip install -U python-openstackclient python-neutronclient`
8. 安装Kolla(controller)   
这里使用的Kolla版本是基于Kolla官方Repo的Stable/Mitaka分支, 但是当前该版本是不提供Ceilometer服务，我们使用目前仍在Review中的[Ceilometer Patch](https://review.openstack.org/#/c/300574/)进行补充，从而开启Ceilometer服务.
    - 克隆Kolla项目:    
    `$ git clone -b stable/mitaka https://github.com/cshuo/kolla.git`  
    - 安装Kolla工具及相关依赖:   
    `$ sudo pip install kolla/`
    - Kolla的配置文件在etc/kolla中, 将之拷贝到/etc/下:    
    `$ cd kolla`     
    `$ sudo cp -r etc/kolla /etc/`


## 配置本地Docker仓库
如果选用All-In-One模式的Openstack, 那么可以选择不安装本地仓库, 但对于多节点Openstack, 本地仓库是必要的.
1. 启动Registry, 推荐使用2.3+版本(controller)     
`$ docker run -d -p 4000:5000 --restart=always --name registry registry:2`
2. 修改docker daemon默认参数(all)     
`$ sudo  vi /etc/default/docker`    
添加以下配置:
```
DOCKER_OPTS="--insecure-registry $CONTROLLER_IP:4000"
```
3. 重启docker      
`$ sudo service docker restart`

## 修改Kolla的配置
Kolla的ansible inventory配置文件指定了那些服务运行在那些主机上，实际部署时需要根据自己的集群情况进行修改，这里我们只有两台虚拟机，一台作为控制节点，运行绝大部分服务，一台作为计算节点，运行计算服务, 部分配置如下:
```
[control]
# These hostname must be resolvable from your deployment host
controller
# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network]
controller
[compute]
compute
[storage]
controller
...
```

## 编译openstack的docker镜像
前面提到Kolla是把Openstack的所有服务都放在容器里，所以在部署前，我们需要编译镜像，默认的编译命令很简单:     
`# kolla-build`      
默认情况下, Kolla是使用CentOS作为基础镜像, 不过本人更习惯使用Ubuntu; 此外，镜像编译完成之后，我们还需要将其推到本地仓库中，所以最后的编译命令是:    
`# kolla-build --base ubuntu --type source --registry $CONTROLLER_IP:4000 --push`    

## 部署Kolla
1. 修改环境变量     
Kolla的所有环境变量都存放在/etc/kolla/下的globals.yml和password.yml中, 首先运行命令:     
`# kolla-genpwd`    
该命令会随机生成密码填充到password.yml中. 然后修改globals.yml,有关这些变量的含义配置文件中详细说明.     
```
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
kolla_internal_vip_address: "$VIP_ADDR"
network_interface: "eth0"
neutron_external_interface: "eth1"
docker_registry: "$CONTROLLER_IP:4000"
...
enable_ceilometer: "yes"
enable_mongodb: "yes"
enable_ceph: "yes"
```

2. 检查部署环境
运行一下命令检查目的主机的配置时候准备完毕:     
`# kolla-ansible prechecks -i <path/to/multinode/inventory/file>`    

3. 开始部署     
`# kolla-ansible deploy -i <path/to/multinode/inventory/file>`     
> 如果你也是使用虚拟机进行部署的话,那么应该是不支持虚拟机的硬件加速的，需要把libvirty的虚拟类型改为qemu, 修改计算节点/etc/kolla/nova-compute/nova.conf, 在[libvirt]模块添加 virt_type=qemu,
```
[libvirt]
...
virt_type=qemu
```

4. 一些有用的工具    
部署完成后，运行以下命令可以生成一个openrc文件(运行openstack CLI所需的环境变量):     
`# kolla-ansible post-deploy`
openrc文件生成之后，使用以下命令可以帮你做一下openstack的初始化工作，包括上传一个glance镜像以及创建几个虚拟网络:
`# source /etc/kolla/admin-openrc.sh`     
`# kolla/tools/init-runonce`

到这里，部署已经全部完成，打开浏览器，输入$VIP_ADDR, 和你的openstack愉快的玩耍吧.


## 错误处理
1. 环境检查错误     
这种错误一般是因为环境的配置不正确(hostname不能解析, 密钥部署错误等),此时一般会有日志输出，根据log信息进行处理.
2. 部署错误
部署时出现的错误，日志信息是在容器中，通过运行一下命令可以查看log:      
`$ docker logs $CONTAINER_NAME`    
根据日志进行调试即可.

由于错误的出现，可能需要多次的部署，而有些错误重新部署是不会进行修正的，所以需要将整个环境进行清理:
`$ tools/cleanup-containers`
`$ tools/cleanup-host`

## 参考资料
- [Kolla部署文档](http://docs.openstack.org/developer/kolla/quickstart.html)
- [Ceilometer补丁](https://review.openstack.org/#/c/300574/)
- [Kolla Ceilometer Bug1](https://bugs.launchpad.net/kolla/+bug/1581565)
- [Kolla Ceilometer Bug2](https://bugs.launchpad.net/kolla/+bug/1582062)
