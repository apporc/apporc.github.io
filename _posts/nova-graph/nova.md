nova: 启动虚拟机流程
=====

![nova](https://oa.eayun.cn/wiki/lib/exe/fetch.php?media=eayunstack:nova:nova.png)

## nova 各服务角色

先说一下 nova 各服务所担当的角色

* nova-api

    nova-api 代表 nova 对外提供服务接口，接受虚拟机的操作指令。无论是从 openstack 的命令行还是 dashboard 界面，只要是虚拟机的
    操作请求最终都会发给 nova-api 服务。这个服务通过 HTTP 协议对外提供 REST API。
* nova-conductor

    nova-conductor 是 nova-compute 之上的一个新服务层。一者它为 nova-compute 代理数据库的操作，即 nova-compute 不直接操作数据库
    而是通过 nova-conductor 的 rpc 方法来读写；二者相对于一些启动、关闭、暂停、唤醒虚拟机等的简单操作，像迁移、重建虚拟机等复杂
    虚拟机操作，会经由 nova-conductor 来协调 nova-compute 共同完成。
* nova-scheduler

    nova-scheduler 是根据特定策略来为某个请求选出一个 nova-compute 来完成任务的服务。策略可以实现负载均衡以及 nova-compute 分组
    等的目的。
* nova-compute

    nova-compute 是直接操作虚拟机的服务，它接到请求之后，通过整合 libvirt, openvswitch/bridge, rbd/iscsi 等的操作来达成请求。

nova 还有以下几个服务，由于在本文中尚不涉及，粗略了解一下：

* nova-cert

    提供证书认证服务的，只在使用 EC2 API 的时候使用。
* nova-consoleauth

    为虚拟机 vnc 控制台访问提供授权的服务。
* nova-novncproxy

    虚拟机 vnc 控制台的代理服务，提供基于 websocket 的浏览器 novnc 客户端。
* nova-objectstore

    接受 S3 格式的 API，将请求转发给 glance。

另外可以在[这里][1]查看官方文档中对于这些 nova 服务角色的描述。

## nova 虚拟机创建流程

创建虚拟机的流程，参照文初的代码调用流程图，图中已将创建虚拟机的调用骨架罗列出来。

1. 从 nova-api 接到创建虚拟机的请求说起。

    请求信息经过匹配之后，最终调用到 nova/api/openstack/compute/servers.py
    中的 create() 方法。这里处理一下请求参数之后，调用 nova/compute/api.py 中的 API.create() 方法。虚拟机的操作基本
    都在 API 下对应一个或多个方法。create() 方法又调用 \_create\_instance() 方法，然后还是处理参数，通过
    \_provision\_instance() 方法在数据库中首先创建了虚拟机相关的记录，包括 instances, block\_device\_mapping 与 quota 等。
    再之后，就通过 nova/conductor/api.py 中的 ComputeTaskAPI.build\_instances() 将请求下发给 nova-conductor。
    build\_instances 又调用 nova/conductor/rpcapi.py 中的 build\_instances()，到这里就是将请求扔进 rabbitmq 消息队列中
    的地方。

    注意看 \_create\_instance() 方法的末尾，在调用完 build\_instances() 方法之后，进行了返回。而 build\_instances() 方法中
    是使用的 cctxt.cast() 方法。 cast() 方法发送出去的 rpc 请求是没有回复的。所以创建虚拟机的请求在 \_create\_instance()
    一步直接返回了，此时虚拟机并没有创建完毕，但是创建的请求已成功发往了 nova-conductor。后续虚拟机的创建情况通过虚
    拟机的状态反映。

    虚拟机的状态目前处在初始状态，就是 \_provision\_instance() 方法中设置的状态，vm_state: BUILDING，task_state: SCHEDULING。

2. nova-conductor 进行 build\_instances 操作。

    nova-conductor 提供了 build\_instances() 这个 rpc 方法，所以它一直在紧切注视着 rabbitmq 消息队列。当看到有一个请求
    指向自己时，它就捡起了这个请求，准备进行 build\_instances() 操作。

    进入 nova/conductor/manager.py 这里，做的事情非常明显，发送一个请求到 nova-scheduler，得到一个选好的运行 nova-compute
    的主机，然后将请求发给 nova-compute。

    选择主机的请求即是 SchedulerClient.select\_destinations()，这个进行的操作也没有什么新鲜，就是通过 nova-scheduler 的
    rpcapi 将请求扔到 rabbitmq 消息队列中。不同的是这次使用的是 cctxt.call() 方法，这个方法是会有消息回复的。回复的内容就是
    选择的 nova-compute 节点名称，由于一次请求创建的虚拟机可能不止一个，所以回复的可能是节点名称的一个列表。

    在收到节点名称之后，build\_instances() 方法开始将请求通过 nova-compute 的 rpcapi 扔到 rabbitmq 队列中。进入
    nova/compute/rpcapi.py 中，看到 build\_and\_run\_instance() 方法也使用了 cctxt.cast() 方法。

3. nova-scheduler  进行 select\_destinations() 操作。

    select\_destinations() 是 nova-scheduler 提供的 rpc 方法。同样 nova-scheduler 注视着 rabbitmq 消息队列，拿起
    select\_destinations 请求，开始执行请求。

    进入 nova/scheduler/manager.py 查看 SchedulerManager 的 select\_destinations 方法。这里调用了 self.driver 的同名方法。
    由于 nova 的配置项 scheduler\_driver 采用的默认值，所以 self.driver 就是 nova.scheduler.filter\_scheduler.FilterScheduler，
    我查了一下，其实也只有仅仅另外一种 driver，就是 nova/scheduler/change.py 中的 ChanceScheduler，不同的 driver 选择主机
    的策略不同。

    进行 nova/scheduler/filter\_scheduler.py 查看 FilterScheduler 的 select\_destinations 方法。跟踪进入 \_schedule() 方法。
    这里进行的事情，基本上就是四步，第一步是从数据库中拿到所有的 nova-compute 节点状态，也就是\_get\_all\_host\_states()
    方法；第二步是依据过滤策略过滤掉一些节点，也就是 get\_filtered\_hosts()； 第三步是对节点列表按一定策略进行排序并选出前面
    几个(具体是几个，取决一个配置项 scheduler\_host\_subset\_size)节点，也就是 get\_weighed\_hosts() 方法；第四步是从最后的
    列表中随机选几个节点。

    将选出的节点列表，回复给 select\_destinations 方法的调用者。

4. nova-compute 从消息队列中捡起请求

    接到 build\_and\_run_instrance() 请求，nova-compute 进入到 nova/compute/manager.py 的 ComputeManager.build\_and\_run\_instance()
    方法。
    build\_and\_run\_instance() 方法，尝试使用 \_build\_and\_run\_instance() 方法进行创建虚拟机。如果重建失败，且捕获了
    RescheduledException，则进行 reschedule 操作，reschedule 的使用场景是虚拟机在某一个 nova-compute 节点 A 上创建失败，将请求转
    往其它 nova-compute 节点重试。

    reschedule 操作，涉及到清理资源。nova-compute 节点 A 上的一些资源，在创建失败的时候会由 build\_and\_run\_instance() 自动清理。
    这里还需要清理的资源就是 neutron 为虚拟机分配的网络资源，所以 reschedule 首先通过 \_cleanup\_allocated\_networks() 方法
    请求 neutron 清理这些资源。等到善后完成，则调用 ComputeTaskAPI.build\_instances() 重新将虚拟机的创建请求发送至 nova-conductor。
    本次创建失败的节点 A 会放置在节点黑名单中，一并发送给 nova-conductor，避免下次又被选到。

    下面进入 \_build\_and\_run\_instance() 方法，看一下创建虚拟机的操作。概括来说，主要分为两个部分的工作，第一部分是虚拟机的周
    边资源的准备，第二部分整合资源借助 libvirt 创建虚拟机。

    * 周边资源准备

        \_build\_and\_run\_instance() 方法调用 \_build\_resources() 来完成这部分工作。也主要有两部分组成，网络的准备与存储的准备。
        分别由 \_build\_networks\_for\_instance() 和 \_prep\_block\_device() 来完成。这两个调用分别请求 neutron 和 cinder 来完成
        相应操作。
        
        在进行网络与存储的准备之前，将虚拟机的状态更改：task\_state 由 SCHEDULING 变为 None，vm\_state 变为 BUILDING。

        \_build\_networks\_for\_instance() 主要是准备控制节点的网络端口等，在网络操作之前将虚拟机的状态调整：vm\_state 设置为
        BUILDING，task\_state 设置为 NETWORKING；
        
        \_prep\_block\_device() 主要是 cinder 端创建卷之类的操作，在操作之前将vm_state设置为 BUILDING，task_state 变为
        BLOCK_DEVICE_MAPPING。这个方法里面最开始生成 block\_device\_mapping 的地方即是根据创建虚拟机的不同磁盘配置，进行不同
        的操作的地方。基于卷、基于镜像创建卷、基于卷快照创建卷等等，创建完还要请求 cinder 端将卷 attach 到虚拟机上。

        关于网络与存储资源准备的详细细节在此不作讨论，需要从以上这两个方法跟踪入对应 neutron 和 cinder 的方法仔细研究。

    * 创建虚拟机

        周边资源准备之后，设置 vm\_state 为 BUILDING，设置 task\_state 为 SPAWNING， 然后进入 nova/virt/libvirt/driver.py 的
        LibvirtDriver.spawn() 方法。 这个方法中，使用 openvswitch、linux-bridge 以及 rbd 和 libvirt 等的底层命令或接口将各资
        源整合起来，完成虚拟机的创建。spawn() 方法中，也分三步:

        * create\_image()，这个方法是处理那些不从 cinder 卷启动的虚拟机，为虚拟机建立临时磁盘。由于临时磁盘
        的后端存在多种类型，每一种类型都有不同的处理方式。如本地磁盘的方式，就需要从 glance 中下载镜像文件；而对于 rbd 则只需要
        使用 rbd 客户端操作对 galnce 镜像进行一次 clone() 操作。

        * \_get\_guset\_xml()，顾名思义，这里是制作虚拟机 xml 的地方。

        * \_create\_domain\_and\_network()，这里将前面准备的周边资源整合起来。

            * 因为前面 neutron 只是准备了控制节点那边的端口，计算节点这边的网络配置还需要 nova-compute 自己来完成，查看
            plug\_vifs() 方法，这里从 openvswitch 的桥接中连出来 veth pair，并引到建立的 linux bridge 之中。最后将虚拟机的
            tap interface 加入到 linux bridge 之中。细节可以跟踪到 nova/virt/libvirt/vif.py 之中去查看。也可以参考我们
            wiki 中[关于 openvswitch 的一篇文章][2]，其中由 nova-compute 完成的那一部分网络操作。

            * \_connect\_volume()，这里将 cinder 准备的卷连接到计算节点上面来。当然并不是所有的卷后端都需要做这一步，如我们使用的 rbd
            就是不需要这一步的。但是像 eqlx 或者其它 iscsi 的后端，就需要这一步。具体不同后端的处理方式，可以进入 nova/virt/libvirt
            /volume.py 之中查看。 rbd 对应的类是 LibvirtNetVolumeDriver，可以看到它并没有实现 connect\_volume() 方法，而
            LibvirtISCSIVolumeDriver 的 connect\_volume() 方法就有很多挂载 iscsi 的内容。

            * 最后 \_create\_domain() 使用 libvirt 发起了虚拟机的创建。

        以上操作完成之后，spawn() 方法并不直接返回，它运行一个 \_wait\_for\_boot() 的方法，一直等待从 libvirt 看到的虚拟机的 power\_state
        变为 RUNNING 为止，才返回。

    以上操作都完成之后 \_build\_and\_run\_instance() 方法将虚拟机的状态落定为: power\_state 为 RUNNING，vm\_state 为 ACTIVE，task\_state 为 None。


 [1]: http://docs.openstack.org/kilo/install-guide/install/apt/content/ch_nova.html
 [2]: https://oa.eayun.cn/wiki/doku.php?id=eayunstack:network:openvswitch
