---
layout: post
title:  "Fuel eqlx/ceph 多后端设计"
date:   2015-03-05 17:25:00
categories: OpenStack
---

# fuel eqlx/ceph multibackend design

### Table of Contents

1. [概述](#section)
2. [问题](#section-1)
    * [多后端配置](#section-2)
    * [高可用与负载均衡](#section-3)
    * [EqualLogic 存储支持](#equallogic-)
3. [解决方法](#section-4)
4. [设计实现](#section-5)
    * [多后端设计](#section-6)
        * [原模块调整](#ceph)
        * [新模块设计](#section-7)
    * [高可用设计](#section-8)
    * [Eqlx 插件设计](#eqlx-)

### 概述

根据我们生产环境的需求，我们需要在我们的 OpenStack 环境中同时使用 EqualLogic 和 Ceph  
两种类型的存储。而 fuel 6.0 对此需求的支持存在问题，故而需要我们做一些开发工作。

### 问题

#### 多后端配置

cinder 多后端即是为 cinder 使用多种不同的存储作为后端，详情参见 [cinder multibackend][1]  
cinder 多后端的配置流程概括来说有如下三步：

1.  在 cinder.conf 配置文件中为每种存储单独配置一个小节来开启多后端，针对我们的情况则  
    是如下两个小节， [cinder_ceph] 和 [eqlx] ：  

        [cinder_ceph]
        # Name this volume backend
        volume_backend_name=cinder_ceph
        # Driver to use for volume creation
        volume_driver=cinder.volume.drivers.rbd.RBDDriver
        # Path to the ceph configuration file
        rbd_ceph_conf=
        # The libvirt uuid of the secret for the rbd_user volumes
        rbd_secret_uuid=
        # The RADOS pool where rbd volumes are stored
        rbd_pool=
        # The RADOS client name for accessing rbd volumes
        rbd_user=

        [cinder_eqlx]
        # Driver to use for volume creation
        volume_driver=cinder.volume.drivers.eqlx.DellEQLSanISCSIDriver
        # Name this volume backend
        volume_backend_name=cinder_eqlx
        # Use thin provisioning for SAN volumes?
        san_thin_provision=
        # IP address of SAN controller
        san_ip=
        # Username for SAN controller
        san_login=
        # Password for SAN controller
        san_password=
        # Group name to use for creating volumes
        eqlx_group_name=
        # Pool in which volumes will be created
        eqlx_pool=

    以上仅列出了必须的一些配置项，额外的配置项可以参考默认的 cinder.conf文件。

2.  在 cinder.conf 中 [DEFAULT] 小节下设置 enabled_backends ，指定开启的后端：

        [DEFAULT]
        # A list of backend names to use. These backend names should
        # be backed by a unique [CONFIG] group with its options (list
        # value)
        enabled_backends=cinder_ceph,cinder_eqlx

3.  使用 cinder 命令行为每一种后端建立一个云硬盘类型：

        cinder --os-username admin --os-tenant-name admin type-create rbd
        cinder --os-username admin --os-tenant-name admin type-key rbd set volume_backend_name=cinder_ceph
        cinder --os-username admin --os-tenant-name admin type-create eqlx
        cinder --os-username admin --os-tenant-name admin type-key eqlx set volume_backend_name=cinder_eqlx

    使用如下命令查看建立的云硬盘类型：

        cinder type-list
        +--------------------------------------+------+
        |                  ID                  | Name |
        +--------------------------------------+------+
        | 2018d398-49bb-499c-87f0-2c3a1cd8a11a | rbd  |
        | 40a196da-f7ad-4215-9053-731633695e7a | eqlx |
        +--------------------------------------+------+

多后端配置完成以后，无论是通过 cinder 命令行还是 horizon 界面建立云硬盘的时候，都可以指定  
云硬盘类型， cinder 命令行是通过指定 --volume-type ：

    cinder create --image-id <image id> --display-name test --volume-type <rbd/eqlx> 1

#### 高可用与负载均衡

在每个控制节点上都启动 cinder-volume ，且各 cinder-volume 都配置了相同了一块存储设备作为后端，  
如果要在不同节点的 cinder-volume 之间实现高可用，需要在各节点的 cinder.conf 中配置相同的 host  
参数。
    
    [DEFAULT]
    # Name of this node.
    host=cinder

在这里我们使用了 cinder 这个值，如果注释掉该选项，该值将默认为节点的主机名。对于三个控制节  
点 (hostA，hostB，hostC) 的情况，设置相同的 host 参数之后：

    cinder service-list
    +------------------+--------------------+------+---------+-------+----------------------------+-----------------+
    |      Binary      |        Host        | Zone |  Status | State |         Updated_at         | Disabled Reason |
    +------------------+--------------------+------+---------+-------+----------------------------+-----------------+
    | cinder-scheduler |       cinder       | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    |  cinder-volume   | cinder@cinder_ceph | nova | enabled |   up  | 2015-04-10T02:10:47.000000 |       None      |
    |  cinder-volume   | cinder@cinder_eqlx | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    +------------------+--------------------+------+---------+-------+----------------------------+-----------------+

注释掉host参数之后则是：

    cinder service-list
    +------------------+--------------------+------+---------+-------+----------------------------+-----------------+
    |      Binary      |        Host        | Zone |  Status | State |         Updated_at         | Disabled Reason |
    +------------------+--------------------+------+---------+-------+----------------------------+-----------------+
    | cinder-scheduler |       hostA        | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    | cinder-scheduler |       hostB        | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    | cinder-scheduler |       hostC        | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    |  cinder-volume   |  hostA@cinder_ceph | nova | enabled |   up  | 2015-04-10T02:10:47.000000 |       None      |
    |  cinder-volume   |  hostB@cinder_ceph | nova | enabled |   up  | 2015-04-10T02:10:47.000000 |       None      |
    |  cinder-volume   |  hostC@cinder_ceph | nova | enabled |   up  | 2015-04-10T02:10:47.000000 |       None      |
    |  cinder-volume   |  hostA@cinder_eqlx | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    |  cinder-volume   |  hostB@cinder_eqlx | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    |  cinder-volume   |  hostC@cinder_eqlx | nova | enabled |   up  | 2015-04-10T02:10:45.000000 |       None      |
    +------------------+--------------------+------+---------+-------+----------------------------+-----------------+

对于注释掉host参数的情况，一块云硬盘 test 如果是由 hostA@cinder_ceph 建立的，那么当  
hostA@cinder_ceph 故障以后，云硬盘 test 将不能管理；而对于设置了相同的 host 参数的情况，  
故障节点 hostA 上创建的云硬盘 test 可以通过剩余的节点 hostB 或 hostC 上的同名 cinder-volume  
来管理。

以上配置相同的 host 参数后，各节点的 cinder-volume 之间即是高可用的，又是负载均衡的。  
如果不配置相同的 host 参数，各节点间也可以负载均衡，但是不能高可用。  

值得注意的是，配置相同的 host 参数在某些 driver 的情况下工作，另外也有一些 driver 则  
不能正常工作，这依赖于 driver 的实现。在我们的情况下，已知 ceph 的 driver 是可以的，  
eqlx 则需要更多的论证和进一步的测试验证。参见： [链接][2] 和 [链接][3]

#### EqualLogic 存储支持

要使用 EqualLogic 存储，只需在 cinder.conf 配置文件中设定 eqlx 相关的一些配置项即可，  
所需配置项在[多后端配置](#section-2)中已指出。

### 解决方法

我们采用 fuel 来部署 OpenStack ，解决以上问题的途径主要是修改 fuel 所使用的 puppet 模块，  
以及开发 fuel 插件。

1. 调整原有 ceph 模块等，使其默认支持 cinder 多后端。
2. 调整原有 cinder 模块为高可用作准备。
3. 开发 eqlx 插件来添加 EqualLogic 存储的支持。

### 设计实现

#### 多后端设计

##### 原模块调整

1.  配置分析

    fuel 原来有 ceph 存储的支持，在部署 openstack 的过程中，它会为 cinder 配置 ceph 后端。  
    但是 fuel 会将 ceph 后端配置为 cinder 的唯一后端，放在 cinder.conf 中的 [DEFAULT]  
    小节下面。详情如下：

        [DEFAULT]
        # Driver to use for volume creation
        volume_driver=cinder.volume.drivers.rbd.RBDDriver
        # Path to the ceph configuration file
        rbd_ceph_conf=
        # The libvirt uuid of the secret for the rbd_user volumes
        rbd_secret_uuid=
        # The RADOS pool where rbd volumes are stored
        rbd_pool=
        # The RADOS client name for accessing rbd volumes
        rbd_user=


    故而需要更改 puppet 模块以完成下面的调整：

    *   新建一个小节 [cinder_ceph]
    *   将 ceph 后端的配置项挪到 [cinder_ceph] 小节之下
    *   在 [cinder_ceph] 小节下指定 volume_backend_name 为 cinder_ceph
    *   在 [DEFAULT] 小节下指定 enabled_backends 列表包含 cinder_ceph

2.  代码设计

    fuel 所使用的 puppet 模块都在 fuel-library 项目中，  
    关于 ceph 的配置在文件 deployment/puppet/openstack/manifests/cinder.pp 中。
    详细内容如下：

        class {'cinder::volume::rbd':
            rbd_pool        => $::ceph::cinder_pool,
            rbd_user        => $::ceph::cinder_user,
            rbd_secret_uuid => $::ceph::rbd_secret_uuid,

    该处使用的资源 cinder::volume::rbd 在 deployment/puppet/cinder/manifests/volume/rbd.pp  
    中定义，其是对 cinder::backend::rbd 资源的一个封装， cinder::backend::rbd 在  
    deployment/puppet/cinder/manifests/backend/rbd.pp中定义。  

    *   cinder::volume::rbd 存在一个问题，不能通过它来指定 volume_backend_name，我们转而直接使用  
        cinder::backend::rbd。实现如下：

            -    class {'cinder::volume::rbd':
            -        rbd_pool        => $::ceph::cinder_pool,
            -        rbd_user        => $::ceph::cinder_user,
            -        rbd_secret_uuid => $::ceph::rbd_secret_uuid,
            -
            +    cinder::backend::rbd {'cinder_ceph':
            +        rbd_pool            => $::ceph::cinder_pool,
            +        rbd_user            => $::ceph::cinder_user,
            +        rbd_secret_uuid     => $::ceph::rbd_secret_uuid,
            +        volume_backend_name => 'cinder_ceph',
            +    }

        cinder::backend::rbd 资源的名称 cinder_ceph 即指定新建 [cinder_ceph] 小节，并将此资源  
        定义的配置项填至该小节下；
        volume_backend_name 选项指定为 cinder_ceph。

    *   使用原有资源 cinder::backends 来指定 enabled_backends。  
        该资源在 deployment/puppet/cinder/manifests/backends.pp 中定义。

            +    class {'cinder::backends':
            +        enabled_backends    => ['cinder_ceph'],

##### 新模块设计

为了创建云硬盘类型，我们需要开发新的模块。

1.  模块设计

    首先，cinder 的 puppet 模块定义了 cinder::type 资源，该资源在  
    deployment/puppet/cinder/manifests/type.pp中定义。cinder::type 允许我们创建新的云硬盘类型。  
    但是我们需要对变量进行处理，对创建过程进行把控，故而要对 cinder::type 资源进行封装。  
    封装后的模块取名 create_vol_types，在  
    deployment/puppet/openstack/manifests/create_vol_types.pp中定义。  

    *   模块接受的参数
        *   云硬盘类型名 set_type
        *   云硬盘与卷后端绑定所依据的键值 set_key
        *   云硬盘对应的 cinder 卷后端名称 set_value

        代码原型如下：

            class openstack::create_vol_types(
            $set_type,
            $set_key,
            $set_value,
            ) {
            ... ...
            }

    *   模块中的变量处理

        cinder::type 资源创建云硬盘也是基于 cinder 命令行来实现的，详情参见  
        文件deployment/puppet/cinder/manifests/type.pp。故而也需要 openrc 中的 auth_url 以及  
        用户名、密码和租户信息。我们从全局变量 $::fuel_settings 中获取这些信息并存在临时变量中。  
        另外，我们云硬盘类型的创建操作只需要进行一次，对于有多个控制节点的环境，我们只需要在  
        其中一个节点上进行即可，我们称该节点为 primary_controller。在此我们基于 $::fuel_settings  
        来判定当前 puppet 运行中节点是否是 primary_controller。  
        代码如下：

            $os_username = $::fuel_settings['access']['user']
            $os_password = $::fuel_settings['access']['password']
            $os_tenant_name = $::fuel_settings['access']['tenant']
            $os_auth_url = "http://${::fuel_settings['management_vip']}:5000/v2.0/"

            $primary_controller = $::fuel_settings['role'] ? { 'primary-controller'=>true, default=>false }

    *   模块中进行的操作

        由于 cinder 命令行需要在 cinder 相关各项系统服务都正常运行的情况下才能执行，而 cinder 服务  
        刚刚启动时尚无法响应命令行的请求。在实际的测试中遇到多次虽然 cinder 服务已经正常启动，但却  
        不响应 cinder 命令行的请求，导致云硬盘类型创建失败。  
        在此我们加入一项操作，来等待 cinder 服务进入工作状态。该操作使用 puppet 内置资源 exec 来实现。  
        代码如下：

            exec {"waiting for cinder service":
                path        => '/usr/bin',
                command     => 'cinder --retries 10 type-list',
                tries       => 10,
                try_sleep   => 1,
                timeout     => 600,
                environment => [
                "OS_TENANT_NAME=${os_tenant_name}",
                "OS_USERNAME=${os_username}",
                "OS_PASSWORD=${os_password}",
                "OS_AUTH_URL=${os_auth_url}",
                ],
                require     => Package['python-cinderclient']
            }

        在这里我们执行命令 cinder --retries 10 type-list 来验证 cinder 服务的响应状态。参数 --retries  
        指定网络超时重试的次数。  
        如果命令成功执行，则认为 cinder 服务已能正常响应命令行的请求。如果执行失败，则进行重试。  
        retries 选项定义了失败重试的次数；  
        try_sleep 定义了两次重试之间休眠的时间；  
        timeout 定义了本操作总的超时时间；  

        之后，就可以调用 cinder::type 资源来进行实际的创建云硬盘类型的操作了。

            cinder::type { $set_type:
                os_username     => $os_username,
                os_password     => $os_password,
                os_tenant_name  => $os_tenant_name,
                os_auth_url     => $os_auth_url,
                set_key         => $set_key,
                set_value       => $set_value,
            }

2.  模块使用

使用新模块创建云硬盘类型，只需要在适当的位置调用 openstack::create_vol_types 资源即可。
示例：

        class {'openstack::create_vol_types':
          set_type  => 'rbd',
          set_key   => 'volume_backend_name',
          set_value => 'cinder_ceph',
        }

然而 OpenStack 的部署是一项复杂的工程，各个操作进行的顺序以及相互之间的依赖要有明确的定义，  
否则将会导致部署失败。这里需要指定的依赖有两项，Class['openstack::cinder'] 和  
Class['cinder::keystone::auth']，分别是 openstack::cinder 资源和 cinder::keystone::auth 资源。  
前述提到，cinder 命令行的执行需要 cinder 各项服务的状态就绪，故而需要 openstack::cinder 资源  
操作完成；  
cinder 与 keystone 的衔接部分则在 cinder::keystone::auth 中配置，衔接成功才能调用cinder命令行。  
manage_volumes 标识了 cinder 的后端类型，当后端类型为 ceph 的时候，即应创建云硬盘类型。
代码如下：

      if ($manage_volumes == 'ceph') {
        class {'openstack::create_vol_types':
          set_type  => 'rbd',
          set_key   => 'volume_backend_name',
          set_value => 'cinder_ceph',
          require   => [Class['openstack::cinder'], Class['cinder::keystone::auth']]
        }

以上代码在deployment/puppet/openstack/manifests/controller.pp文件中，该文件是 fuel 部署控制  
节点的模块。

#### 高可用设计

fuel 原来在配置 ceph 后端的时候，在 [DEFAULT] 小节下指定了 host，其值为 rbd:${rbd_pool}，  
在我们的情况下即为 rbd:volumes。该操作在deployment/puppet/cinder/manifests/backend/rbd.pp中：

    cinder_config {
    "${name}/volume_backend_name":              value => $volume_backend_name;
    "${name}/volume_driver":                    value => 'cinder.volume.drivers.rbd.RBDDriver';
    "${name}/rbd_ceph_conf":                    value => $rbd_ceph_conf;
    "${name}/rbd_user":                         value => $rbd_user;
    "${name}/rbd_pool":                         value => $rbd_pool;
    "${name}/rbd_max_clone_depth":              value => $rbd_max_clone_depth;
    "${name}/rbd_flatten_volume_from_snapshot": value => $rbd_flatten_volume_from_snapshot;
    "${name}/host":                             value => "rbd:${rbd_pool}";
    }

第一，需要去掉 rbd 模块中对 host 选项的更改。代码更改如下：

        cinder_config {
        "${name}/volume_backend_name":              value => $volume_backend_name;
        "${name}/volume_driver":                    value => 'cinder.volume.drivers.rbd.RBDDriver';
        "${name}/rbd_ceph_conf":                    value => $rbd_ceph_conf;
        "${name}/rbd_user":                         value => $rbd_user;
        "${name}/rbd_pool":                         value => $rbd_pool;
        "${name}/rbd_max_clone_depth":              value => $rbd_max_clone_depth;
        "${name}/rbd_flatten_volume_from_snapshot": value => $rbd_flatten_volume_from_snapshot;
    -   "${name}/host":                             value => "rbd:${rbd_pool}";
        }


第二，需要在 cinder 配置模块中另行指定 [DEFAULT] 小节的 host 选项。在  
deployment/puppet/openstack/manifests/cinder.pp中添加如下一段：

    +  cinder_config {
    +    'DEFAULT/host':   value => 'cinder';
    +  }

#### Eqlx 插件设计

关于 Fuel 插件代码结构，请参见 [fuel plugin development][4]
本设计只阐述与 eqlx 插件本身相关的部分。

*   tasks.yaml：

    定义 eqlx 插件只在控制节点上执行，执行的时间为整个 OpenStack 环境部署完成后。
    
        - role: ['controller']
        stage: post_deployment

*   metadata.yaml：

    定义 eqlx 插件的名称版本等信息。  
    名称：cinder_eqlx  
    界面标题：Cinder and Eqlx integration  
    兼容的 fuel 版本：6.0.1  
    支持的发行版：CentOS  
    支持的部署模式：ha 和非 ha 均可  

*   environment_config.yaml：

    定义本插件在 fuel web UI 上展示的配置项。  
    san_ip：Dell EqualLogic 磁盘阵列的 IP 地址，对应界面上的“IP”配置项。  
    san_login：访问阵列的用户名，对应界面上的“Username”配置项。  
    san_password：访问阵列的密码，对应界面上的“Password”配置项。  
    eqlx_group_name：卷组的名称，对应界面上的“Volume Group Name”配置项。  
    eqlx_pool：存储池的名称，对应界面上的“Pool Name”配置项。  

*   deployment_scripts/puppet/plugin_cinder_eqlx/manifests/init.pp：

    puppet 模块定义插件工作内容。

    *   基于 $::fuel_settings 全局变量判断插件部署之前的 puppet 部署是否成功  
        如果不成功，则打印日志，放弃插件部署。

            if $::fuel_settings {
                ... ...
            }
            else {
            notify {'Empty fuel_settings, plugin deployment canceled.':}
            }

    *   依据 $::fuel_settings 判断是否为 primary_controller

            $primary_controller = $::fuel_settings['role'] ? { 'primary-controller'=>true, default=>false }

    *   依据 $::fuel_settings 来判断部署环境是否开启了 ceph 后端  
        如果开启了 ceph 则将 enabled_backends 设置为既包含 cinder_ceph，  
        又包含 cinder_eqlx；否则只包含 cinder_eqlx即可。
        
            if $::fuel_settings['storage']['volumes_ceph'] {
                $enabled_backends = ['cinder_ceph','cinder_eqlx']
            }
            else {
                $enabled_backends = ['cinder_eqlx']
            }

    *   使用 cinder::backend::eqlx 资源配置 eqlx 后端，配置完后通知 cinder-volume 服务重启。

            cinder::backend::eqlx { 'cinder_eqlx':
                san_ip                    => $::fuel_settings['cinder_eqlx']['san_ip'],
                san_login                 => $::fuel_settings['cinder_eqlx']['san_login'],
                san_password              => $::fuel_settings['cinder_eqlx']['san_password'],
                eqlx_group_name           => $::fuel_settings['cinder_eqlx']['eqlx_group_name'],
                eqlx_pool                 => $::fuel_settings['cinder_eqlx']['eqlx_pool'],
                san_thin_provision        => true,
                volume_backend_name       => 'cinder_eqlx',
                eqlx_use_chap             => false,
                eqlx_cli_timeout          => 30,
                eqlx_cli_max_retries      => 5,
            } ~>
            service { $::cinder::params::volume_service:
            }

    *   使用 cinder::backends 资源配置 enabled_backends，配置完通知 cinder-volume 服务重启。

            class { 'cinder::backends':
                enabled_backends  => $enabled_backends,
                notify            => Service[$::cinder::params::volume_service],
            }

    *   判断是否为 primary_controller，如果是则安装 python-cinderclient 包，  
        然后创建 eqlx 云硬盘类型，由于插件部署是在整个OpenStack完成部署后，所以该操作只指定对于  
        cinder::backend::eqlx 资源的依赖即可。  

            if $primary_controller {
                package {'python-cinderclient':
                } ->
                cinder::type { 'eqlx':
                os_username     => $::fuel_settings['access']['user'],
                os_password     => $::fuel_settings['access']['password'],
                os_tenant_name  => $::fuel_settings['access']['tenant'],
                os_auth_url     => "http://${::fuel_settings['management_vip']}:5000/v2.0/",
                set_key         => 'volume_backend_name',
                set_value       => 'cinder_eqlx',
                require         => Cinder::Backend::Eqlx['cinder_eqlx'],
                }
            }

 [1]: http://docs.openstack.org/admin-guide-cloud/content/multi_backend.html
 [2]: https://bugzilla.redhat.com/show_bug.cgi?id=1132722
 [3]: https://ask.openstack.org/en/question/55268/ha-recovering-from-cinder-node-loss-with-ceph-backend/
 [4]: /openstack/2015/03/06/fuel-plugin-development.html
