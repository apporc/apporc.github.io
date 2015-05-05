---
layout: post
title:  "OpenStack Nova: 手动删除虚拟机"
date:   2015-05-05 11:56:00
categories: OpenStack
---

调试 OpenStack 环境或线上故障时，经常出现数据不完整情况。典型的就是故障状态的虚拟机无论是通过 nova delete 还是 nova
force-delete 都无法删除。有一个手动删除虚拟机的操作步骤有一定必要性。以下总结一个删除的流程。

注意：

* 手动删除虚拟机需操作敏感数据，危险性高，操作需谨慎，如无必要尽量不要使用该方式。

* 以下步骤只适用于一种场景：neutron + cinder ( ceph / eqlx )，其余场景需自行适配。

手动删除虚拟机流程：

1. 删除 consoleauth 服务为其存在 memcache 中的 token。

    要在每个控制节点上都执行一次：

        echo 'delete <vm_uuid>' | nc <controller_ip> 11211

2. 清理计算节点上虚拟机相关资源：

    * 在控制节点上，获取虚拟机所在计算节点以及其名称：

            nova show <vm_uuid>

        OS-EXT-SRV-ATTR:host 或 OS-EXT-SRV-ATTR:hypervisor_hostname 显示其所在节点  
        OS-EXT-SRV-ATTR:instance_name 显示虚拟机名称

    * 前往计算节点：

        * 确认名称为<vm_name>的虚拟机其 uuid 与 <vm_uuid> 一致：

                virsh dumpxml <vm_name>

        * 如果虚拟机仍开启，则关闭虚拟机：

                virsh destroy <vm_name>

        * 删除虚拟网卡与网桥：
            * 在控制节点查询数据库，登录 nova 数据库：

                    select network_info from instance_info_caches where instance_uuid='<vm_uuid>';

            获取其中 devname 字段后 11 位或者 ovs_interfaceid 前 11 位。  

            示例：  
            devname: tapf6e315dd-d9  
            ovs_interfaceid: f6e315dd-d984-4ff1-9705-b778cd0153aa
            取出来的11位为 f6e315dd-d9

            虚拟网卡与虚拟网桥的名称都基于这11位字符：  

            虚拟机的虚拟网卡<vm_vif>为：tapf6e315dd-d9  
            虚拟网卡对应网桥为<vm_vbr>：qbrf6e315dd-d9  
            网桥对应网卡名为<>：qvbf6e315dd-d9  
            openvswitch port名<ovs_port>为：qvof6e315dd-d9  

            * 删除虚拟网卡，网桥等

                    brctl delif <vm_vbr> <vm_vif>
                    ip link set <vm_vbr> down
                    brctl delbr <vm_vbr>
                    ovs-vsctl --if-exists del-port br-int <ovs_port>

                示例：

                    brctl delif qbrf6e315dd-d9 tapf6e315dd-d9
                    ip link set qbrf6e315dd-d9 down
                    brctl delbr qbrf6e315dd-d9
                    ovs-vsctl --if-exists del-port br-int qvof6e315dd-d9

        * 删除存储连接
            * iscsi:
                在控制节点，登陆 nova 数据库查看 iscsi 连接信息：

                    select connection_info from block_device_mapping where instance_uuid='<vm_uuid>';

                保存 target\_iqn 和 target\_portal，在计算节点进行如下操作：

                    iscsiadm -m node -T <target_iqn> -p <target_portal> --op update -n node.startup -v manual
                    iscsiadm -m node -T <target_iqn> -p <target_portal> --logout
                    iscsiadm -m node -T <target_iqn> -p <target_portal> --op delete

            * rbd:
                不需要操作


        * 删除虚拟机文件夹：

                rm -rf /var/lib/nova/instances/<vm_uuid>
                rm -rf /var/lib/nova/instances/<vm_uuid>_resize
                rm -rf /var/lib/nova/instances/<vm_uuid>_del

        * undefine 虚拟机
            
                virsh undefine <vm_uuid>

3. 通过 cinder api 删除卷

    在如下命令的输出中 os-extended-volumes:volumes\_attached 中标识了该虚拟机的volume\_uuid

        nova show <vm_uuid>

    注：boot from image 的虚拟机没有 volume  

    使用如下命令删除卷

        nova volume-detach <vm_uuid> <volume_uuid>
        nova volume-delete <volume_uuid>

    对于 detach 无法成功的情况，需要操作 cinder 数据库强制删除，其步骤不在本文档讨论范围。

4. 通过 neutron api 删除虚拟机对应 port

        nova interface-list <vm_uuid>

    记录所列出的　port_id，然后通过如下命令删除 port

        neutron port-delete <port_id>

    如果删除无法成功的，需要操作 neutron 数据库强制删除，其步骤不在本文档讨论范围。

5. 修改 nova 数据库

    下面修改数据库：

    * 更新 instances 表，将 vm\_state 改为 deleted，power\_state 改为 0 (界面上对应的状态即 NOSTATE)

            update instances set vm_state='deleted',power_state=0 where uuid='<vm_uuid>';

    * 更新 quota 值：

        使用如下命令查看 project\_id，tenant\_id 即 project\_id

            nova show vm_uuid


        使用如下命令虚拟机所占用的内存与 CPU

            nova flavor-show

        更新数据库

            update quotas set  hard_limit=hard_limit+1 where project_id='<project_id>' and resource='instances';
            update quotas set  hard_limit=hard_limit+x where project_id='<project_id>' and resource='cores';
            update quotas set  hard_limit=hard_limit+x where project_id='<project_id>' and resource='ram';

    * 相关数据表更新：

            update instance_faults set deleted=id where instance_uuid='<vm_uuid>';
            update instance_info_caches set deleted=id where instance_uuid='<vm_uuid>';
            update security_group_instance_association set deleted=id where instance_uuid='<vm_uuid>';
            update block_device_mapping set deleted=id where instance_uuid='<vm_uuid>';
            update fixed_ips set deleted=id where instance_uuid='<vm_uuid>';
            update virtual_interfaces set deleted=id where instance_uuid='<vm_uuid>';
            update intances set deleted=id where instance_uuid='<vm_uuid>';



6. glance 镜像

    如果是shelved状态的虚拟机，需要通过 glance api 删除快照  

    使用如下命令查看需要删除的镜像：

        nova image-list 

    Server 一栏对应 为 vm\_uuid 的那个镜像需要删除

        nova image-delete image_uuid

