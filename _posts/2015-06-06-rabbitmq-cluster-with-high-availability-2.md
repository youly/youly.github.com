---
layout: post
title: 使用rabbitmq提供高可用性队列服务(2)
category: 分布式
tags: [queue,rabbitme,ha]
---

[上篇文章](/2015/05/30/rabbitmq-cluster-with-high-availability-1/)提到了rabbitmq的一些概念和特性，本文承接上文，讲解如何通过docker部署rabbitmq节点。

###准备环境

rabbitmq集群有几点要求：

* 节点拥有相同的Erlang、Rabbitmq版本
* 节点拥有相同的Erlang Cookie（Erlang虚拟机使用cookie来决定节点间能否通信）
* 节点网络处于同一个网段

为了满足以上要求，本文使用docker来部署rabbitmq节点。

####安装docker

由于 docker daemon 使用到了linux内核的一些特性，因此docker的安装对于操作系统类型和版本有一些要求。

对于 centos，要求版本6.5及以上，内核版本2.6.32-431及以上。可参考[文档说明](https://docs.docker.com/installation/centos/)。

当然 Mac OS X 是不支持的。如果一定要在mac上玩，可以下载[boot2docker](https://github.com/boot2docker/osx-installer/releases)，为运行docker定制的虚拟机。

####制作docker镜像

本文将在centos下安装和运行rabbitmq，docker安装完后执行以下命令下载centos镜像：

    docker pull centos


接下来制作rabbitmq镜像，以便可以启动多个容器：

    # 启动一个 centos 环境下运行 bash 程序的容器
    docker run -it centos /bin/bash

    # 安装 Erlang
    yum -y install wget
    mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
    wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
    yum -y update
    mkdir packages && cd packages
    wget http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
    wget http://rpms.famillecollet.com/enterprise/remi-release-6.rpm
    rpm -Uvh remi-release-6*.rpm epel-release-6*.rpm
    yum install -y erlang

    # 安装 Rabbitmq
    wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.2.2/rabbitmq-server-3.2.2-1.noarch.rpm
    rpm --import http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
    yum -y install rabbitmq-server-3.2.2-1.noarch.rpm
    
    # 启用 rabbitmq 管理插件
    rabbitmq-plugins enable rabbitmq_management

    # 设置 Erlang Cookie
    RUN echo "ERLANGCOOKIE" > /var/lib/rabbitmq/.erlang.cookie
    RUN chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
    RUN chmod 400 /var/lib/rabbitmq/.erlang.cookie
    chown -R rabbitmq:rabbitmq /var/lib/rabbitmq

    # 安装其他一些工具
    yum -y install net-tools vim

    yum install -y passwd openssh-clients openssh-server
    ssh-keygen -f /etc/ssh/ssh_host_rsa_key
    ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key
    echo 'root:root' | chpasswd
    /bin/sed -i 's/.*session.*required.*pam_loginuid.so.*/session optional pam_loginuid.so/g' /etc/pam.d/sshd

以上命令(可以写成一个Dockerfile)执行完后，一个rabbitmq环境就搭建好了，control + p && control +q 退出当前容器，执行以下命令，提交 image：

    # 查看当前所有容器
    docker ps -a

    # 选择刚才制作好的container，执行
    docker commit {container id} rabbitmq

    # 查看image
    docker images

###启动rabbitmq节点

启动主机名为rbq1的节点：

    docker run -it -h rbq1 rabbitmq /bin/bash
    [root@rbq1 /]# rabbitmq-server -detached

启动主机名为rbq2的节点：

    docker run -it -h rbq2 rabbitmq /bin/bash
    [root@rbq2 /]# rabbitmq-server -detached

修改主机rbq1、rbq2，hosts文件分别新增以下两行

    172.17.0.4      rbq1
    172.17.0.6      rbq2

###配置rabbitmq集群

我们以 rbq1 为主节点，将 rbq2 作为salve加入到rbq1集群:

    [root@rbq2 /]# rabbitmqctl join_cluster rabbit@rbq1
    [root@rbq2 /]# rabbitmqctl  start_app
    Starting node rabbit@rbq2 ...
    ...done.
    [root@rbq2 /]# rabbitmqctl cluster_status
    Cluster status of node rabbit@rbq2 ...
    [{nodes,[{disc,[rabbit@rbq1,rabbit@rbq2]}]},
     {running_nodes,[rabbit@rbq1,rabbit@rbq2]},
      {partitions,[]}]
      ...done.

在 rbq1 上执行：

    [root@rbq1 /]# rabbitmqctl cluster_status
    Cluster status of node rabbit@rbq1 ...
    [{nodes,[{disc,[rabbit@rbq1,rabbit@rbq2]}]},
     {running_nodes,[rabbit@rbq2,rabbit@rbq1]},
      {partitions,[]}]
      ...done.

###配置集群ha

    [root@rbq1 /]# rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}'
    Setting policy "ha-all" for pattern "^ha\\." to "{\"ha-mode\":\"all\"}" with priority "0" ...
    ...done.
    [root@rbq1 /]# rabbitmqctl list_policies
    Listing policies ...
    /   ha-all  all ^ha\\.  {"ha-mode":"all"}   0
    ...done.

以上指令配置了所有以ha开头的队列在所有节点中镜像一份。rabbitmq镜像模式有以下几种：

![rabbitmq-ha](/assets/images/rabbitmq-ha.png)

###参考
1、[Rabbitmq Clustering Guide](https://www.rabbitmq.com/clustering.html)

2、[Rabbitmq Highly Available Queues](https://www.rabbitmq.com/ha.html)

3、[Distributed RabbitMQ brokers](https://www.rabbitmq.com/distributed.html)






