手动删除虚拟机流程

适用场景

非本地存储
非 nova-network

1. glance
    如果是shelved状态的虚拟机，需要通过 glance api 删除快照
    nova image-list 
    Server 一栏对应 虚拟机的 uuid 的那个需要删除
    nova image-delete image_uuid


2. 数据库
    instance 表，将 vm_states 改为 deleted，power_state NOSTATE

    instance_id_mappings no delete
    待查instance_actions_events no delete
    待查instance_actions no delete

    instance_faults
    instance_info_caches
    instance_system_metadata
    security_group_instance_association
    待查block_device_mapping
    待查fixed_ips
    待查virtual_interfaces

    instances

    以上将deleted 改为 id

    更新 quota 值：
    quotas 表，　project_id, instance -1 core -x ram -x
    nova show vm_uuid，tenant_id对应project_id
    nova flavor-show 获取 ram, vcpus

3. cinder

    nova show vm_uuid
    os-extended-volumes:volumes_attached中标识了该虚拟机的volume

    boot from image 的虚拟机没有 volume (image uuid 未查到)

    volume 是否要删除需要用户输入

    nova volume-detach vm_uuid volume_uuid
    nova volume-delete volume_uuid

4. neutron
    nova interface-list vm_uuid
    neutron port-delete

5. 计算节点清理

    nova show vm_uuid (with admin)

    OS-EXT-SRV-ATTR:host 或 OS-EXT-SRV-ATTR:hypervisor_hostname 显示其所在节点
    OS-EXT-SRV-ATTR:instance_name 显示虚拟机名称

    到相应节点
    ensure the instance_name's uuid
    virsh destroy instance_name


vifs 
我们的环境中虚拟机网卡都是类型：ovs_hybrid_plug
该信息在instance_info_caches中。

devname 

utils.execute('brctl', 'delif', br_name, v1_name,
                run_as_root=True)
utils.execute('ip', 'link', 'set', br_name, 'down',
                run_as_root=True)
utils.execute('brctl', 'delbr', br_name,
                run_as_root=True)

_ovs_vsctl(['--', '--if-exists', 'del-port', bridge, dev])


no client support firewall_rule at now

disconnect_volume

if iscsi:

iscsiadm -m node -T
iqn.2001-05.com.equallogic:0-fe83b6-44ed650d4-eadb0c75e9755391-volume-cbbdf96f-c157-498c-be53-83b9d220ec3d -p
172.16.102.251:3260 --op update -n node.startup -v manual

 iscsiadm -m node -T
 iqn.2001-05.com.equallogic:0-fe83b6-44ed650d4-eadb0c75e9755391-volume-cbbdf96f-c157-498c-be53-83b9d220ec3d -p
 172.16.102.251:3260 --logout

 iscsiadm -m node -T
 iqn.2001-05.com.equallogic:0-fe83b6-44ed650d4-eadb0c75e9755391-volume-cbbdf96f-c157-498c-be53-83b9d220ec3d -p
 172.16.102.251:3260 --op delete

if rbd:
    nothing

delete instance files

rm -rf /var/lib/nova/instances/vm_uuid
rm -rf /var/lib/nova/instances/vm_uuid_resize
rm -rf /var/lib/nova/instances/vm_uuid_del

serial_console release port
usually type is file
if tcp, need to  memory port, warning

virsh undefine

6. consoleauth 删除　token

echo 'delete vm_uuid' | nc controllers_ip 11211



删除命令：

1. 删除 consoleauth 服务为其存在 memcache 中的 token。

要在每个控制节点上都执行一次：

    echo 'delete <vm_uuid>' | nc <controller_ip> 11211

2. 清理计算节点上虚拟机相关资源：

    在控制节点上，获取虚拟机所以计算节点以及其名称：

        nova show <vm_uuid>

    OS-EXT-SRV-ATTR:host 或 OS-EXT-SRV-ATTR:hypervisor_hostname 显示其所在节点
    OS-EXT-SRV-ATTR:instance_name 显示虚拟机名称

    前往计算节点：

    * 确认相应名称的虚拟机其 uuid 与 vm_uuid 一致：

        virsh dumpxml <vm_name>

    * 如果虚拟机仍开启，则关闭虚拟机：

        virsh destroy <vm_name>

    * 删除虚拟网卡与网桥：
        * 控制节点，登录 nova 数据库：

            select network_info from instance_info_caches where instance_uuid='<vm_uuid>';

        获取其中 devname 字段后 11 位或者 ovs_interfaceid 前 11 位：
        示例：
        devname: tapf6e315dd-d9
        ovs_interfaceid: f6e315dd-d984-4ff1-9705-b778cd0153aa

        取出来为 f6e315dd-d9

        则虚拟机的虚拟网卡对为：tapf6e315dd-d9
        虚拟网卡对应网桥为：qbrf6e315dd-d9
        网桥对应网卡名为：qvbf6e315dd-d9
        openvswitch port名为：qvof6e315dd-d9

        * 删除虚拟网卡，网桥等

        命令：
            
            brctl delif qvbf6e315dd-d9 tapf6e315dd-d9
            ip link set qvbf6e315dd-d9 down
            brctl delbr qvbf6e315dd-d9

            ovs-vsctl --if-exists del-port br-int qvof6e315dd-d9

    * 删除存储连接
        * iscsi:
            查看 iscsi 连接信息：

                select connection_info from block_device_mapping where instance_uuid='<vm_uuid>';

            保存 target_iqn 和 target_portal：

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

    nova show <vm_uuid>
    os-extended-volumes:volumes_attached中标识了该虚拟机的<volume_uuid>

    注：boot from image 的虚拟机没有 volume (image uuid 未查到)

    volume 是否要删除需要用户确认

    nova volume-detach <vm_uuid> <volume_uuid>
    nova volume-delete <volume_uuid>

    对于 detach 无法成功的情况，需要操作 cinder 数据库强制删除，其步骤不在本文档讨论范围。

4. 通过 neutron api 删除

        nova interface-list <vm_uuid>

    记录所列出的　port 的 id，然后通过如下命令删除 port

        neutron port-delete <port_id>

    如果删除无法成功的，需要操作 neutron 数据库强制删除，其步骤不在本文档讨论范围。

5. 数据库

    下面修改数据库：

    instance 表，将 vm_state 改为 deleted，power_state 0 (NOSTATE)

    instance_id_mappings no delete
    待查instance_actions_events no delete
    待查instance_actions no delete

    instance_faults
    instance_info_caches
    instance_system_metadata
    security_group_instance_association
    待查block_device_mapping
    待查fixed_ips
    待查virtual_interfaces

    instances

    以上将deleted 改为 id

    更新 quota 值：
    quotas 表，　project_id, instance -1 core -x ram -x
    nova show vm_uuid，tenant_id对应project_id
    nova flavor-show 获取 ram, vcpus

6. glance
    如果是shelved状态的虚拟机，需要通过 glance api 删除快照
    nova image-list 
    Server 一栏对应 虚拟机的 uuid 的那个需要删除
    nova image-delete image_uuid

