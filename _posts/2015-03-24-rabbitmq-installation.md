---
layout: post
title:  "RabbitMQ：安装与基本操作"
date:   2015-03-23 11:52:00
categories: rabbitmq
---

#### Table of Contents

1. [安装](#section)
2. [配置简介](#section-1)
    * [配置文件](#section-2)
    * [用户管理](#section-3)
    * [防火墙](#section-4)
    * [system limits](#system-limits)
3. [RabbitMQ 集群](#rabbitmq-)
    * [手动创建集群](#section-5)
    * [重启集群](#section-6)
    * [破坏集群](#section-7)
    * [自动创建集群](#section-8)
4. [fuel 中的 RabbitMQ](#fuel--rabbitmq)

### 安装

rabbitmq的安装分两部分，rabbitmq-server自身的安装以及其依赖erlang的安装。

* rabbitmq-server：
    rabbitmq-server的安装包fedora系统安装源有提供，epel也有提供。
    * fedora系统源：
        提供的包版本往往比较老旧，比如现在fedora20的源提供的版本为3.1.5。
    * [EPEL][1]：
        提供的版本要新得多，目前epel-7提供的包版本为3.3.5。
    * rabbitmq官网：
        提供的最新版本为[3.5][2]

* erlang：
    erlang包的几个来源是：
    * [EPEL][1]：
        Red Hat/fedora官方包，包被分解为多个小包，版本不太新。
    * [Erlang Solutions][3]：
        包版本最新，提供yum安装源安装和单独下载安装的包。
    * [Erlang from RabbitMQ][4]：
        经过裁减的，只包含rabbitmq所需功能的包。

安装完之后，通过service rabbitmq-server start来启动服务。默认配置下即可
成功运行rabbitmq-server。  
查看 rabbitmq 状态：  

    rabbitmqctl status

进一步的配置请继续向下。

### 配置简介

#### 配置文件
RabbitMQ的配置有三个来源：环境变量，配置文件和运行时参数。

* 环境变量  
    RabbitMQ的环境变量定义在/etc/rabbitmq/rabbitmq-env.conf中,位置不可指定。

    > 值得一提的是，RabbitMQ对环境变量的解析顺序为：  
    1.首先是在shell中形如RABBITMQ_var_name的变量  
    2.其次是在配置文件rabbitmq-env.conf中var_name变量  
    3.最后是RabbitMQ中预定义的默认值  

* 配置文件  
    RHEL/CentOS下配置文件默认即是/etc/rabbitmq/rabbitmq.config。  
    该配置文件的位置可配，指定环境变量RABBITMQ_CONFIG_FILE  
    或者在/etc/rabbitmq/rabbitmq-env.conf中定义CONFIG_FILE都可以。  
    在rabbitmq的日志中有如下一行显示配置文件路径：

    `config file(s) : /etc/rabbitmq/rabbitmq.config`

* 运行时参数
    还有另外一类特定的配置参数，要么它需要保持在所有rabbitmq节点上保持一致，
    要么它在运行时会发生变化，这类参数可以使用rabbitmqctl来配置。

        rabbitmqctl set_parameter {-p vhost} component_name name value
        rabbitmqctl clear_parameter {-p vhost} component_name name
        rabbitmqctl list_parameters {-p vhost}

#### 用户管理
首次启动RabbitMQ时，RabbitMQ会初始化自己的数据库，并建立如下资源：

* 一个默认虚拟主机,名称为"/"

    查看虚拟主机： `rabbitmqctl list_vhosts`

* 一个guest用户，密码为guest，全权访问"/"虚拟主机

    查看所有用户： `rabbitmqctl list_users`

guest用户是不安全的，建议删除或者更改其密码。

* 删除： `rabbitmqctl delete_user guest`
* 更改密码： `rabbitmqctl change_password guest <password>`

为了安全，通常不使用 guest 用户，所以需要创建自定义用户，示例：

    rabbitmqctl add_user nova <password>

#### 防火墙
根据配置不同，RabbitMQ会使用很多tcp端口：

    4369 (epmd), 25672 (Erlang distribution)
    5672, 5671 (启用了 或者 未启用 TLS 的 AMQP 0-9-1)
    15672 (如果管理插件被启用)
    61613, 61614 (如果 STOMP 被启用)
    1883, 8883 (如果 MQTT 被启用)

当配置了RabbitMQ集群后，还有额外端口：  
在/etc/rabbitmq/rabbitma.config中inet_dist_listen_min和inet_dist_listen_max所
限定的端口。

#### system limits

rabbitmq 会维持大量的网络连接，所以系统允许打开的最大文件数需要调整。  
可以通过 ulimit -n 为单个用户环境设定该值，  
或者通过 /etc/security/limits.conf 在系统范围内更改。  

### RabbitMQ 集群

本节示例创建一个五节点的 rabbitmq 集群，名称分别为：  

* rabbit
* rabbit1
* rabbit2
* rabbit3
* rabbit4

在开始下面配置之前，需要将以上主机名写入各节点 /etc/hosts 文件。  
只有主机名可正常解析的情况下，以下配置才可继续。

#### 手动创建集群

1.  同步 erlang cookie
    rabbitmq 使用 erlang 语言写的，erlang 节点之间要通信需要有相同的 cookie。  
    所谓 cookie 是由数字和字母组成一个字符串，长度任意。  
    创建 rabbitmq 集群的第一步即是同步 cookie。  
    Linux 系统的 erlang cookie 通常在 /var/lib/rabbitmq/.erlang.cookie。  
    只需要确保每个 rabbitmq 节点上的该文件一致即可。  
    最简单的做法就是在某一个节点上生成 cookie，然后复制到其它节点上。  
    rabbitmq-server 初次启动时会自动生成 cookie，你也可以自已指定 cookie。  

    如果之前未曾启动，在 rabbit 上启动 rabbitmq 来生成 cookie (手动方式)：
        
        rabbit$ rabbitmq-server -detached

    从 rabbit 上把　cookie 复制到其它节点：

        for i in `seq 1 4`
        do 
            scp  /var/lib/rabbitmq/.erlang.cookie rabbit$i:/var/lib/rabbitmq/.erlang.cookie
        done

2.  启动独立节点
    rabbitmq 集群的构建是通过将已有节点重新配置的方式来加入集群的。  
    所以需要在每个节点上把 rabbitmq-server 启动一次：

        rabbit1$ rabbitmq-server -detached
        rabbit2$ rabbitmq-server -detached
        rabbit3$ rabbitmq-server -detached
        rabbit4$ rabbitmq-server -detached


3.  配置集群
    在除 rabbit 之外的节点上，调整 rabbitmq。

        rabbitmqctl stop_app
        rabbitmqctl join_cluster rabbit@rabbit
        rabbitmqctl start_app

    查看一下集群的状态，

        rabbit$ rabbitmqctl cluster_status

        Cluster status of node 'rabbit@rabbit' ...
        [{nodes,[{disc,['rabbit@rabbit1','rabbit@rabbit2',
                        'rabbit@rabbit3','rabbit@rabbit4',
                        'rabbit@rabbit']}]},
        {running_nodes,['rabbit@rabbit1','rabbit3@rabbit2',
                        'rabbit@rabbit3','rabbit1@rabbit4',
                        'rabbit@rabbit']},
        {cluster_name,<<"rabbit@rabbit">>},
        {partitions,[]}]
        ...done.

    以上 nodes 一行是本集群所有的节点名称，running_nodes 则是运行状态的节点名称。

#### 重启集群

在每个节点上执行如下命令停止 rabbitmq

    rabbitmqctl stop

再要重新启动，调用

    rabbitmq-server -detached

**注意：** 对于所有节点都关闭了的集群，重新启动的时候：  

1.  最后关闭的节点必须最先启动。  
2.  如果最后关闭的节点没有先启动，其它节点启动时会等待 30 秒，然后失败放弃。
3.  如果最后关闭的节点由于某种原因启动不了，其它节点启动时可以使用如下命令删除掉。

        rabbitmqctl forget_cluster_node rabbit@rabbit

4.  如果集群故障，所有节点同时停掉，例如同时断电的情况，所有节点都认为其它节点是  
    最后停掉的。 这种情况下可以使用 force_boot 命令来强制启动节点。

    **注意：**force_boot命令在 rabbitmq 3.4 版本以上才有。目前 epel7 源中的版本 3.3.5 还没有。

    在没有 force_boot 命令的情况下，尝试同时启动各个节点也可以重新启动集群。

#### 破坏集群

对于每个集群节点，使用如下命令可以使其脱离集群，从而破坏整个集群：

    rabbitmqctl stop_app
    rabbitmqctl reset
    rabbitmqctl start_app

特殊的，当某一个节点 (如rabbit4) 已经连接不上，此时可以在在线的节点上执行如下命令：

    rabbitmqctl forget_cluster_node rabbit4@rabbit

之后，如果 rabbit4 系统又恢复了，想要重新加入启动 rabbitmq，会出错：

    rabbitmqctl start_app ( or rabbitmq-server -detached )
    {error,{inconsistent_cluster,"Node 'rabbit4@rabbit4' thinks it's clustered with node 'rabbit@rabbit', but 'rabbit@rabbit' disagrees"}}

此时需要先重置 rabbit4 后再重启：

    rabbitmqctl reset ( or rm -rf /var/lib/rabbitmq/mnesia/rabbit
    rabbitmqctl start_app ( or rabbitmqctl -detached )


#### 自动创建集群

除了使用上面手动的方式构建集群之外，也可以通过指定配置文件来构建集群，只需要在
/etc/rabbitmq/rabbitmq.config 中写入节点的列表即可：

    [
    ...
    {rabbit, [
            ...
            {cluster_nodes, {['rabbit@rabbit1', 'rabbit@rabbit2', 'rabbit@rabbit3'], disc}},
            ...
    ]},
    ...
    ].

该种方式存在的问题是：　只对于从未启动过的或重置过的 rabbitmq 节点可用，其它的不行。


#### fuel 中的 rabbitmq

1.  fuel 中的 rabbitmq 由 pacemaker 管理

    其控制脚本路径为 /usr/lib/ocf/resource.d/mirantis/rabbitmq-server

    多节点环境下，rabbitmq 节点有 Master 和 Slave 两种身份。确认 rabbitmq-server 状态：

        crm resource list 

    在其中找到关于 rabbitmq-server 的信息，类似：

        Master/Slave Set: master_p_rabbitmq-server [p_rabbitmq-server]
            Masters: [ node-3.domain.tld ]
            Slaves: [ node-1.domain.tld node-2.domain.tld ]

    如果有某个节点上服务已停止，则有节点状态为 Stopped：

        Master/Slave Set: master_p_rabbitmq-server [p_rabbitmq-server]
            Masters: [ node-1.domain.tld ]
            Slaves: [ node-3.domain.tld ]
            Stopped: [ node-2.domain.tld ]

    或者某个节点上服务出现故障，则有节点状态为 FAILED：

        Master/Slave Set: master_p_rabbitmq-server [p_rabbitmq-server]
            p_rabbitmq-server  (ocf::mirantis:rabbitmq-server):        FAILED 
            Masters: [ node-11.eayun.test ]
            Slaves: [ node-10.eayun.test ]

    你也可以使用如下命令来查看状态：

        crm resource show master_p_rabbitmq-server

2.  使用 rabbitmqctl 查看集群状态

    使用 pacemnaker 提供的工具查看 rabbitmq 的状态，所查看到的状态仅仅是 rabbitmq-server  
    服务的状态，更准确的状态需要通过 rabbitmq 自身提供的工具。

    查看单一节点的状态 (进程 pid，版本，在运行的程序，占用内存，侦听的端口等详细信息)：

        rabbitmqctl status

    查看集群状态 (集群中某节点的运行状态)：

        rabbit$ rabbitmqctl cluster_status

        Cluster status of node 'rabbit@rabbit' ...
        [{nodes,[{disc,['rabbit@rabbit1','rabbit@rabbit2',
                        'rabbit@rabbit3','rabbit@rabbit4',
                        'rabbit@rabbit']}]},
        {running_nodes,['rabbit@rabbit1','rabbit3@rabbit2',
                        'rabbit@rabbit3','rabbit1@rabbit4',
                        'rabbit@rabbit']},
        {cluster_name,<<"rabbit@rabbit">>},
        {partitions,[]}]
        ...done.

    返回结果中，nodes 一行是本集群所有的节点名称，running_nodes 则是运行状态的节点名称。  
    不在 running_nodes 中而在 nodes 中的节点即为故障的节点。  
    抛开所有节点不谈，只要 partitions 不为空，则集群状态已出现故障。

3.  配置部分

    * 端口

        fuel 所部署的 OpenStack中 haproxy 作为 rabbitmq 的负载均衡器，它监听 5672  
        端口，然后将请求转发至5673。然而就目前观察，OpenStack 中并没有服务使用了 haproxy  
        提供的该 5672 端口的服务，各 OpenStack 服务都是直接连接节点的 5673 端口。

    0     0 ACCEPT     tcp  --  *      *       172.16.100.2         0.0.0.0/0            multiport sports 4369,5672,15672,41055,55672,61613 /* 003 remote rabbitmq  */
98358 5901K ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            multiport ports 4369,5672,5673,41055 /* 106 rabbitmq  */
    * system limits

        Fuel 在 /etc/security/limits.conf 中设置了 system limits：
        最大文件数的 soft limits 102400，hard limits 为 112640。

    * vhosts 和 user

        只有一个默认虚拟主机，列出虚拟机主机：

            # rabbitmqctl list_vhosts
            Listing vhosts ...
            /
            ...done

        只有一个用户 nova，所有 OpenStack 服务都使用该用户连到 rabbitmq，列出用户：

            # rabbitmqctl list_users
            Listing users ...
            nova    [administrator]
            ...done.

 [1]:http://fedoraproject.org/wiki/EPEL/FAQ#howtouse
 [2]:https://www.rabbitmq.com/install-rpm.html
 [3]:https://www.erlang-solutions.com/downloads/download-erlang-otp
 [4]:https://www.rabbitmq.com/releases/erlang/
