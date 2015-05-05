---
layout: post
title:  "OpenStack Nova：虚拟机启动流程"
date:   2015-03-19 17:35:00
categories: OpenStack
---


#### Table of Contents

1. [概述](#section)
2. [流程](#section-1)
    * [horizon/cli](#horizoncli)
    * [nova-api](#nova-api)
    * [nova-conductor](#nova-conductor)
    * [nova-scheduler](#nova-scheduler)
    * [nova-compute](#nova-compute)

### 概述

这几天从创建虚拟机这条线缕了一下nova的代码，将虚拟机启动的流程整理在此，
更详细的代码解读将在另外的博客中逐步总结。  
由于整理得比较仓促，细节处可能有误，本文会随时更新。

### 流程

#### horizon或CLI
1. dashboard或者CLI获取用户的凭证，并通过REST API向keystone服务发起权限验证，
keystone 验证通过之后会返回一个auth_token，后续请求都将使用该auth_token。
2. dashboard或者CLI将用户输入整理一下，调用REST API向nova-api发送创建虚拟机的
请求。

#### nova-api

* nova-api接到创建虚拟机的请求
* 使用请求中包含的auth_token向keystone发送请求作权限验证。
* 整理请求中的参数，形成create_kwargs
    * admin_password 
        获取request中的password
    * access_ips  
        将request中的access_ips，ipv4或ipv6取出来了整理进create_kwargs。
    * availabilityzone  
        获取request中的os-availability-zone,填进create_kwargs。
    * block device mapping
        即block device mapping v2，获取request中的block_device_mapping_v2信息，
        依其形成BlockDeviceDict填进create_kwargs。
    * block device mapping v1
        即legacy device mapping，获取request中的block_device_mapping信息，直接
        填写进create_kwargs，同时在create_kwargs中将legacy_bdm置真，便于后续流程。
        此项与上一项对应的处理不同版本的block device mapping数据。
    * config drive  
        获取request中的config_drive信息填写进create_kwargs。
        config_drive允许用户在一个数据盘中存放信息被cloud-init读取，cloud-init依
        据这些信息在虚拟机初次启动时作一些自动化的配置。
    * disk config  
        获取request中的api disk config，形如OS-DCF:diskConfig，处理为internal disk
        config，填写进create_kwargs。
    * keypairs create  
        获取request中的key_name填写进create_kwargs。
    * multiple create  
        获取request中的min_count，max_count，以及return reservation id。
        该选项实现一个请求创建多个instance的功能，自动创建的instance的名称可以nova.conf中作定义。
        这三个参数分别指定了一次创建虚拟机数目的最小值，最大值和是否要返回reservation_id。
        是不是要返回reservation id由return_reservation_id决定。该值是multiple create的
        虚拟机id上限的意思。
    * personality  
        获取request中的personality，整理填写进create_kwargs中的injected_files，
        injected_files中的文件将被放进虚拟机中。
    * scheduler_hints  
        获取request中的os:scheduler_hints或OS-SCH-HNT:scheduler_hints填写进create_kwargs中的
        scheduler_hints。代码中说其意义是传递给scheduler的键值对。推测为nova-scheduler所需。
    * security_groups  
        获取request中的security_groups参数，整理填写进create_kwargs。其中有一个list(set())操作，
        去掉重复指定的值。
    * user_data  
        获取request中的user_data填写进create_kwargs。
    * image_uuid，requested_networks，flavor_id

* 将create_kwargs作为参数传给computeapi发起创建。
* computeapi对请求作鉴权，nova鉴权的配置文件默认在/etc/nova/policy.json，该文件里定义了nova中哪
    些角色可以作哪些操作。
* 校验所有参数的合法性
    * 检查multiple create下，不能指定ip或者neutron network port
    * reservation id是否新建
    * security_groups,min_count,max_count,block_device_mapping设定默认值
    * 如果没有指定flavor,设定默认的flavor
    * 获取相应image或者volume的信息，确保其支持auto_disk_config，即自动调整磁盘大小
    * 处理availabilityzone信息，检查policy是否许可forced_host,forced_node
    * 检查请求中所提到的各资源是否存在，是否合法
        * availabilityzone是否存在
        * flavor是否存在
        * user_data长度是否合法，并解码
        * 对于create和rebuild进行特殊检查
            * metadata quota
            * injected_files quota
            * image的状态等信息校验
        * 获取security_groups，是否存在
        * 访问network api检查requested_networks合法性
        * 从image信息中获取kernel和ramdisk信息
        * 校验config drive.
        * 获取keypair校验
        * 校验root device name，即在虚拟机中磁盘的设备路径，例如/dev/vda
        * numa topology
        * 从flavor获取pci request info
        * 通知network api配置sriov ports
        * 以上处理之后构建新参数列表base_options
        * 将image中的一些信息更新进base_options
        * 返回base_options
    * check_and_transform_bdm检查所有block device mappings的合法性，其root device有没有重合之类
        对于image则基于image建立block device mapping。同一volume不可attach给两个instance。
    * get_requested_instance_group检查scheduler_hints是否合法。
    * provision_instance
        * 将实例的vm_state设置为BUILDING，task_state设置为SCHEDULING。
        * check_num_instances_quota数据库中获取quota，quota检查，计算quota，cpu,RAM，该quota对象更改后提交。
        * 针对可创建的虚拟机数目循环在数据库中创建。
        * 调用security_groups api处理安全组。
        * 在数据库中创建相应block_device_mapping条目。
        * 对于指定了instance group的实例，更新instancegroup相关quota，更新instancegroup数据库表。
        * 将虚拟机创建事件通过rpc广播出去，便于ceilometer等记录。
    * 记载虚拟机状态变化record_action_start，创建InstanceActions表
    * 将虚拟机创建命令通过rpc发送给nova-conductor。
    * 命令发送成功，nova-api的流程即已走完，此时界面按钮或者CLI将返回。

#### nova-conductor

* nova-conductor从消息队列中拾起创建虚拟机请求
* 从数据库中读取相关资源的信息，整理请求参数为json格式。
* 信息整合之后，通过scheduler client向scheduler发送rpc请求，让scheduler选择合适的nova-compute节点。
* scheduler选好nova-compute节点之后，通过消息队列返回结果。
* conductor通过compute rpc api发送给相应nova-compute节点。

#### nova-scheduler

* nova-scheduler从消息队列中拾起节点选择的请求
* 发起rpc广播schedule事件开始
* scheduler有一系列Filter，在不同层面定义过滤策略。将所有节点通过这些策略进行过滤。
* 为过滤之后的节点列表依据一定的策略作排序。
* 从排序后的节点列表中自前向后选择一定数目的节点，所取节点数目取决于scheduler的配置项scheduler_host_subset_size。
* 在最后选择出的节点列表中，随机选择节点。
* 发起rpc广播schedule事件结束。

#### nova-compute

* nova-compute从消息队列中拾起虚拟机创建请求
* 为实例加锁，并新建线程进行创建虚拟机操作，以便释放rpc的workder线程。
* 每个compute节点有最大并行创建数限制，该限制由eventlet.semaphore实现。
* 设置虚拟机task_state由SCHEDULING变为None，vm_state变为BUILDING。
* 发起rpc广播虚拟机create事件开始。
* 锁定虚拟机创建所需的资源。
* 为虚拟机创建准备虚拟网络和虚拟磁盘等资源。
    * 创建虚拟网络
        * 将vm_state设置为BUILDING，task_state设置为NETWORKING。
        * 获取macs和dhcp配置信息。
        * 通过network_api发送请求为虚拟机创建虚拟网络。
    * 创建虚拟磁盘
        * 将vm_state设置为BUILDING，task_state变为BLOCK_DEVICE_MAPPING。
        * 虚拟机的磁盘设备有这几类：卷，快照，镜像和以及待创建的空盘。
            * 对于卷，挂载到compute节点。
            * 对于快照，调用cinder依据快照创建新卷，挂载到compute节点。
            * 对于镜像，调用cinder依据镜像创建新卷，挂载到compute节点。
            * 对于所需特定大小的空卷，通知cinder依大小创建并挂载到compute节点。
* 虚拟机所需资源准备就绪以后，设置vm_state为BUILDING，设置task_state为SPAWNING。
* 准备虚拟机配置路径，组装xml。
* 调用特定hypervisor的driver(libvirt，xenapi，hyperv，vmware)来创建虚拟机。
* 发起rpc广播虚拟机create事件结束。
* 将虚拟机power state设置为1，将vm state设置为active，将task_state置为None。
* 如果虚拟机创建失败，允许retry的情况下会通过重发给scheduler进行reschedule操作，重发创建过程。
