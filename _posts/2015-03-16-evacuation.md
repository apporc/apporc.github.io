Evacuation
==================

#### 目录

1. [概述](#section)
2. [命令](#section-1)
3. [开发工作](#section-2)
3. [参考](#section-3)


#### 概述

OpenStack为故障的compute节点提供Evacuation功能，也就是故障迁移。
当某一个compute节点出现故障，我们会需要将其上的虚拟机恢复运行。
由于故障节点的修复时间是不定的，第一时间的选择往往是将虚拟机迁移
到其它的节点上先运行起来。

#### 命令

1. 首先需要确认哪台计算节点故障，可以通过nova service-list
查看。出现故障的节点其nova-compute状态为down。注意：除此之外，还需要
前往相应的故障节点，确认其确实已经宕机，假如只是 nova-compute 服务
故障，而其上虚拟机都正常运行，是不应当进行故障迁移的。

将故障节点A上的虚拟机迁移出去的命令为：

    nova host-evacuate A

节点故障时集体迁出虚拟机，由于可能存在的存储与网络压力，不一定能完美
迁移，对于需要迁移的虚拟机列表，管理员应逐个确认是否迁移成功，迁移不成功
的虚拟机，再单独进行迁移处理，单独迁移某一虚拟机的命令如下：
单纯指定只迁移某个虚拟机 V：

    nova evacuate V

单独迁移虚拟机 V 至 B 节点：

    nova evacuate V B

查看集群中有哪些compute节点，可以使用命令

    nova host-list

2. 所有虚拟机都迁移好之后，可以去恢复故障节点。
故障节点恢复以后，在启动后需要人工做一些善后处理：
查看原虚拟机的残余信息：

    virsh list --all

对于列表中存在的虚拟机，需要手工清除，以虚拟机 instance-xxx 为例，需要执行：

    virsh undefine instance-xxx

另外，还需要清理虚拟机残留在磁盘上的文件：

    cd /var/lib/nova/instances
    ls

需要将该目录下所有以 uuid 命名的文件夹全部删除，例如
70a94ba5-7b38-4bd7-9421-11d25b90c81b 和
70a94ba5-7b38-4bd7-9421-11d25b90c81b_resize 等。

    rm 70a94ba5-7b38-4bd7-9421-11d25b90c81b  70a94ba5-7b38-4bd7-9421-11d25b90c81b_resize


#### 开发工作

将evacuate的命令集成进我们的运维脚本中，作一层包装。

#### 参考
[nova_cli_evacuate](http://docs.openstack.org/user-guide-admin/content/nova_cli_evacuate.html)

#### 注意

evacuate 命令对于不同存储类型的虚拟机存在不同的处理方式，需要留心：
1. 对于以卷为磁盘(boot from volume)的虚拟机， evacuate 操作时旧有虚拟机中的
数据(内存除外)得到保存
2. 对于以镜像为磁盘(boot from image)的虚拟机， evacuate 操作相当于是在新节点上
新建一个虚拟机，所以旧有虚拟机中的数据一概消失。
