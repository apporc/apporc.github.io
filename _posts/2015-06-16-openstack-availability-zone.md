OpenStack Host Aggregates 和 Availability Zone 的使用

## 简介

OpenStack 可以支持庞大节点数量的集群，达到这一点除了依赖其自身优秀的架构之外，还
有赖于其对 Region，Cells，Host Aggregates 和 Availability Zone 的支持。
Region 与 Cells 是通过把多个 nova 部署整合在一起来扩展 OpenStack 集群，在这里暂不
讨论。这里主要说一下如何使用 Host Aggregates 和 Availability Zone 来扩展单一的一
个 nova 部署。

## 概念与命令

Aggregates 和 Availability Zone 都是将计算节点进行分组的单位，可以将某些计算节点
划入一个 Aggregate，把某些 Aggregate 划入一个 Availability Zone。

nova 命名中关于 aggregate 的操作有：

    aggregate-add-host          Add the host to the specified aggregate.
    aggregate-create            Create a new aggregate with the specified
                                details.
    aggregate-delete            Delete the aggregate.
    aggregate-details           Show details of the specified aggregate.
    aggregate-list              Print a list of all aggregates.
    aggregate-remove-host       Remove the specified host from the specified
                                aggregate.
    aggregate-set-metadata      Update the metadata associated with the
                                aggregate.
    aggregate-update            Update the aggregate's name and optionally
                                availability zone.
关于 availability zone 的有：

    availability-zone-list      List all the availability zones.

细查这些命令的帮助手册，即可了解到：

1. 一个 availability zone 是若干 aggregate 的集合。
2. availability zone 对普通用户可见，aggregate 则仅对 admin 角色可见。
3. availability zone 无法独立于 aggregate 来操作，仅有的一条命令只是用来列出 zone 的列表。availability zone 的创建与销毁都是暗含在 aggregate 的操作中。
4. 同一个节点可以同时被划入多个 aggregate 或 availability zone 中。
5. 使用上，availability zone 是通过 nova boot 命令的 --availability-zone 参数来发挥作用；aggregate 则是通过其上定义的 metadata 配合 nova-scheduler。

## 操作示例

新建一个名称为 test1 的 aggregate，并将它划入名称为 test 的
availability zone，如果 test 不存在，则新建 test。

    $ nova aggregate-create test1 test
    +-----+-------+-------------------+-------+--------------------------+
    | Id  | Name  | Availability Zone | Hosts | Metadata                 |
    +-----+-------+-------------------+-------+--------------------------+
    | 283 | test1 | test              |       | 'availability_zone=test' |
    +-----+-------+-------------------+-------+--------------------------+

也可以新建一个不属于任何 availability zone 的 aggregate

    $ nova aggregate-create test2

某个 OpenStack 部署都有一个默认的 availability zone 名称为 nova，新建一个 test3
并将其加入 nova

    $ nova aggregate-create test3 nova

列出已创建的 aggregate 列表

    $ nova aggregate-list
    +-----+-------+-------------------+
    | Id  | Name  | Availability Zone |
    +-----+-------+-------------------+
    | 283 | test1 | test              |
    | 287 | test2 | -                 |
    | 289 | test3 | nova              |
    +-----+-------+-------------------+

列出已创建的 availability zone 列表

    $ nova availability-zone-list
    +-----------------------+----------------------------------------+
    | Name                  | Status                                 |
    +-----------------------+----------------------------------------+
    | internal              | available                              |
    | |- node-5.eayun.com   |                                        |
    | | |- nova-conductor   | enabled :-) 2015-06-17T08:14:41.000000 |
    | | |- nova-consoleauth | enabled :-) 2015-06-17T08:14:44.000000 |
    | | |- nova-scheduler   | enabled :-) 2015-06-17T08:14:44.000000 |
    | | |- nova-cert        | enabled :-) 2015-06-17T08:14:44.000000 |
    | |- node-6.eayun.com   |                                        |
    | | |- nova-conductor   | enabled XXX 2015-06-15T08:13:26.000000 |
    | | |- nova-consoleauth | enabled XXX 2015-06-15T08:13:18.000000 |
    | | |- nova-scheduler   | enabled XXX 2015-06-15T08:13:16.000000 |
    | | |- nova-cert        | enabled XXX 2015-06-15T08:13:22.000000 |
    | |- node-8.eayun.com   |                                        |
    | | |- nova-conductor   | enabled :-) 2015-06-17T08:14:41.000000 |
    | | |- nova-consoleauth | enabled :-) 2015-06-17T08:14:44.000000 |
    | | |- nova-scheduler   | enabled :-) 2015-06-17T08:14:44.000000 |
    | | |- nova-cert        | enabled :-) 2015-06-17T08:14:46.000000 |
    | nova                  | available                              |
    | |- node-4.eayun.com   |                                        |
    | | |- nova-compute     | enabled :-) 2015-06-17T08:14:41.000000 |
    | |- node-7.eayun.com   |                                        |
    | | |- nova-compute     | enabled :-) 2015-06-17T08:14:45.000000 |
    +-----------------------+----------------------------------------+

**注意**：由于 test 这个 availability zone 的所有 aggregate (test1) 尚未包含任何的
计算节点，所以 availability zone 列表中不予列出。internal 也是 nova 内部运作
需要的 availability zone，不必关注，下文会略去这部分输出。

添加一个节点至 test1，计算节点名称可以通过 `nova host-list` 查询

    $ nova aggregate-add-host test1 node-7.eayun.com
    Host node-7.eayun.com has been successfully added for aggregate 283
    +-----+-------+-------------------+--------------------+--------------------------+
    | Id  | Name  | Availability Zone | Hosts              | Metadata                 |
    +-----+-------+-------------------+--------------------+--------------------------+
    | 283 | test1 | test              | 'node-7.eayun.com' | 'availability_zone=test' |
    +-----+-------+-------------------+--------------------+--------------------------+

test1 中新加了一个计算节点之后，availability zone 的列表中就出现了 test

    $ nova availability-zone-list
    +-----------------------+----------------------------------------+
    | Name                  | Status                                 |
    +-----------------------+----------------------------------------+
    | test                  | available                              |
    | |- node-7.eayun.com   |                                        |
    | | |- nova-compute     | enabled :-) 2015-06-17T08:19:56.000000 |
    | nova                  | available                              |
    | |- node-4.eayun.com   |                                        |
    | | |- nova-compute     | enabled :-) 2015-06-17T08:19:56.000000 |
    +-----------------------+----------------------------------------+

同一个计算节点可以被同时划入两个 aggregate

    $ nova aggregate-add-host test2 node-7.eayun.com
    Host node-7.eayun.com has been successfully added for aggregate 287
    +-----+-------+-------------------+--------------------+----------+
    | Id  | Name  | Availability Zone | Hosts              | Metadata |
    +-----+-------+-------------------+--------------------+----------+
    | 287 | test2 | -                 | 'node-7.eayun.com' |          |
    +-----+-------+-------------------+--------------------+----------+

将节点 node-7.eayun.com 移出 test2

    $ nova aggregate-remove-host test2 node-7.eayun.com

删除 test2

    $ nova aggregate-delete test2

在 test 中启动一个虚拟机

    $ nova boot --flavor m1.tiny --image c7a8f232-d24d-4dd9-9b8b-16c6949a0dc4\
    --availability-zone test zone_test

    $ nova show zone_test
    +--------------------------------------+----------------------------------------------------------+
    | Property                             | Value                                                    |
    +--------------------------------------+----------------------------------------------------------+
    | OS-DCF:diskConfig                    | MANUAL                                                   |
    | OS-EXT-AZ:availability_zone          | test                                                     |
    | OS-EXT-SRV-ATTR:host                 | node-7.eayun.com                                         |
    | ...                                  | ...                                                      |
    +--------------------------------------+----------------------------------------------------------+

为 test1 设置 metadata "abc=1"

    $ nova aggregate-set-metadata test1 abc=1
    Metadata has been successfully updated for aggregate 283.
    +-----+-------+-------------------+--------------------+-----------------------------------+
    | Id  | Name  | Availability Zone | Hosts              | Metadata                          |
    +-----+-------+-------------------+--------------------+-----------------------------------+
    | 283 | test1 | test              | 'node-7.eayun.com' | 'abc=1', 'availability_zone=test' |
    +-----+-------+-------------------+--------------------+-----------------------------------+

通过以上命令的反馈，我们可以看到 aggregate 与 availability zone 的关系其实也是通
过 aggregate 的 metadata 来体现的。
通过开启 nova-scheduler 的特定 scheduler driver，并设置相应 flavor 的
extra_spec 为 abc=1 可以使得基于相应 flavor 创建的虚拟机都被限定到 test1 之中，
详细的使用方法参见下文。

## 结合我们的实际情况

由于计算节点数量庞大，而网络和存储的承载能力有限，使得我们必须将特定数目的
计算节点规划于某一交换机与存储集群之上，避免与其它交换机或存储集群发生联系。
一方面我们需要 aggregate 这样的单位来对计算节点进行分组，另外一方面我们还需要
指定能够访问某一 aggregate 的租户列表。

达成以上要求，有三个方案可以选择：

#### 方案一：使用 availability zone

使每一个 availability zone 对应一个 aggregate，用户在创建虚拟机的时候自行选择将
虚拟机建立在哪个 zone 下面。

##### 操作方法：

horizon界面上创建虚拟机时允许用户选择 availability zone，
或者使用 `nova boot --availability-zone` 指定也可以。

##### 优点：

不需要运维人员作复杂的设定，部署 OpenStack 环境时一次配置，后续不必再动。

##### 缺点：

不能针对 zone 设置有权限的 tenant 列表，所有的 zone 对用户都是可见的，用
      户可以任意选择 zone。

#### 方案二：配合使用 flavor 与 aggregate 的 metadata

为每个 aggregate 设定关键字，例如某一 aggregate 的所有计算节点都属于机架一，则
可以为该 aggregate 设定 "rack=1" 的 metadata；相应的设定某一 tenant 可以使用的
flavor 都有 "rack=1" 这一 extra_spec，那么相应 tenant 所创建的虚拟机就只会走到
我们安排的机架上去。

##### 操作方法：

1.  确认 nova-scheduler 支持 AggregateInstanceExtraSpecsFilter

    前往控制节点，查看 /etc/nova/nova.conf

        $ grep AggregateInstanceExtraSpecsFilter /etc/nova/nova.conf
        scheduler_default_filters=AggregateMultiTenancyIsolation,AggregateInstanceExtraSpecsFilter,RetryFilter,AvailabilityZoneFilter,RamFilter,CoreFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter

2. 部署 OpenStack 完成后，对计算节点进行分组

    创建一个 aggregate 为 "rack01" 代表机架一上的计算节点，其所属 availability zone
    设为默认的 nova

        $ nova aggregate-create rack01 nova
        +-----+--------+-------------------+-------+--------------------------+
        | Id  | Name   | Availability Zone | Hosts | Metadata                 |
        +-----+--------+-------------------+-------+--------------------------+
        | 291 | rack01 | nova              |       | 'availability_zone=nova' |
        +-----+--------+-------------------+-------+--------------------------+

    把机架一上的节点添加进 rack01

        $ nova aggregate-add-host rack01 node-4.eayun.com
        Host node-4.eayun.com has been successfully added for aggregate 291
        +-----+--------+-------------------+--------------------+--------------------------+
        | Id  | Name   | Availability Zone | Hosts              | Metadata                 |
        +-----+--------+-------------------+--------------------+--------------------------+
        | 291 | rack01 | nova              | 'node-4.eayun.com' | 'availability_zone=nova' |
        +-----+--------+-------------------+--------------------+--------------------------+

    为 rack01 设置 metadata "rack=1"

        $ nova  aggregate-set-metadata rack01 rack=1
        Metadata has been successfully updated for aggregate 291.
        +-----+--------+-------------------+--------------------+------------------------------------+
        | Id  | Name   | Availability Zone | Hosts              | Metadata                           |
        +-----+--------+-------------------+--------------------+------------------------------------+
        | 291 | rack01 | nova              | 'node-4.eayun.com' | 'availability_zone=nova', 'rack=1' |
        +-----+--------+-------------------+--------------------+------------------------------------+

3. 当有新的用户要使用 OpenStack 中的资源时，为其创建租户并作配置

    创建一个新租户 new_tenant

        $ keystone tenant-create --name new_tenant --description "authorized to use rack01"
        +-------------+----------------------------------+
        |   Property  |              Value               |
        +-------------+----------------------------------+
        | description |     authorized to use rack01     |
        |   enabled   |               True               |
        |      id     | e68be5f7aa32429b906aaa70833ed949 |
        |     name    |            new_tenant            |
        +-------------+----------------------------------+

    创建一个新用户 new_user，属于租户 new_tenant，用户密码随意

        $ keystone user-create --name new_user --tenant new_tenant --pass abc123
        +----------+----------------------------------+
        | Property |              Value               |
        +----------+----------------------------------+
        |  email   |                                  |
        | enabled  |               True               |
        |    id    | 978b5a51fdec43c9964468345990babd |
        |   name   |             new_user             |
        | tenantId | e68be5f7aa32429b906aaa70833ed949 |
        | username |             new_user             |
        +----------+----------------------------------+

    为新用户创建 flavor (flavor 应为一个系列，在此只以 tiny 为例)

        $ nova flavor-create --is-public false rack01.tiny auto 64 1 1
        +--------------------------------------+-------------+-----------+------+-----------+------+-------+-------------+-----------+
        | ID                                   | Name        | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
        +--------------------------------------+-------------+-----------+------+-----------+------+-------+-------------+-----------+
        | 1b965f74-de4a-4530-95e1-9d8f2488c1ed | rack01.tiny | 64        | 1    | 0         |      | 1     | 1.0         | False     |
        +--------------------------------------+-------------+-----------+------+-----------+------+-------+-------------+-----------+
        $ nova flavor-access-add rack01.tiny e68be5f7aa32429b906aaa70833ed949
        +--------------------------------------+----------------------------------+
        | Flavor_ID                            | Tenant_ID                        |
        +--------------------------------------+----------------------------------+
        | 1b965f74-de4a-4530-95e1-9d8f2488c1ed | e68be5f7aa32429b906aaa70833ed949 |
        +--------------------------------------+----------------------------------+

    为 flavor 设置 extra_spec

        $ nova flavor-key rack01.tiny set aggregate_instance_extra_specs:rack=1
        $ nova flavor-show rack01.tiny
        +----------------------------+----------------------------------------------+
        | Property                   | Value                                        |
        +----------------------------+----------------------------------------------+
        | OS-FLV-DISABLED:disabled   | False                                        |
        | OS-FLV-EXT-DATA:ephemeral  | 0                                            |
        | disk                       | 1                                            |
        | extra_specs                | {"aggregate_instance_extra_specs:rack": "1"} |
        | id                         | 1b965f74-de4a-4530-95e1-9d8f2488c1ed         |
        | name                       | rack01.tiny                                  |
        | os-flavor-access:is_public | False                                        |
        | ram                        | 64                                           |
        | rxtx_factor                | 1.0                                          |
        | swap                       |                                              |
        | vcpus                      | 1                                            |
        +----------------------------+----------------------------------------------+

    之后，用户 new_user 基于 rack01.tiny 去创建虚拟机，虚拟机就只会跑到机架一上去。

##### 注意：

1. 需要保证用户 new_user 有权使用的所有 flavor 都必须有 aggregate_instance_extra_specs:rack=1，否则他/她使用别的 flavor 会将虚拟机建到机架一之外去。
2. 除了为某个 tenant 都建一系列 flavor 这种作法之外，也可以只建一个系列的 flavor，比如 rack01.tiny，rack01.small，rack01.medium，rack01.large，然后通过 `nova flavor-access-add` 授权需要使用机架一的 tenant。

##### fuel 中需要作的更改：

1. nova 默认开启 AggregateInstanceExtraSpecsFilter
2. 将默认创建的 flavor 的 is-public 设为 false

##### 优点：

比较合理地实现了 tenant 对 aggregate 的权限

##### 缺点：

运维人员需要进行较复杂的操作

#### 方案三：使用 nova-scheduler 的 AggregateMultiTenancyIsolation

使用 AggregateMultiTenancyIsolation 可以直接限定能够访问某一 aggregate 的
tenant 列表

##### 操作方法：

1. 确认 AggregateMultiTenancyIsolation 已开启

    前往控制节点

        $ grep AggregateMultiTenancyIsolation /etc/nova/nova.conf
        scheduler_default_filters=AggregateMultiTenancyIsolation,AggregateInstanceExtraSpecsFilter,RetryFilter,AvailabilityZoneFilter,RamFilter,CoreFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter

2. 为 aggregate 设定有权访问的 tenant

    默认所有的 tenant 都有权使用所有的 aggregate，虽然用户看不到 aggregate 这个单位
    ，但是其虚拟机还是能跑到各个 aggregate 中去的。所以此处作限定，意味着必须为所有
    的 aggregate 都作设定，不作设定的 aggregate 即是大家公用的。

    设定 rack01 这个 aggregate 只能由租户 e68be5f7aa32429b906aaa70833ed949 来使用

        $ nova aggregate-set-metadata rack01 filter_tenant_id=e68be5f7aa32429b906aaa70833ed949
        Metadata has been successfully updated for aggregate 291.
        +-----+--------+-------------------+---------------------+-----------------------------------------------------------------------------------------+
        | Id  | Name   | Availability Zone | Hosts               | Metadata                                                                                |
        +-----+--------+-------------------+---------------------+-----------------------------------------------------------------------------------------+
        | 291 | rack01 | nova              | 'node-10.eayun.com' | 'availability_zone=nova', 'filter_tenant_id=e68be5f7aa32429b906aaa70833ed949', 'rack=1' |
        +-----+--------+-------------------+---------------------+-----------------------------------------------------------------------------------------+

##### 注意：

一个 aggregate 的 filter_tenant_id 只能包含一个 id，该值无法以列表的形态
指定，即一个 aggregate 只能由某一个租户独享。但并不意味着机架是独享的，因为一个
机架上的主机是可以同时被划到多个 aggregate 中去的。

##### fuel 中需要作的更改：

1. nova 默认开启 AggregateMultiTenancyIsolation 

##### 优点：

实现了 tenant 对 aggregate 的权限

##### 缺点：

灵活性低，每当有一个新用户加入，就需要为其新建一个 aggregate，然后相应机架上的主机
加入到 aggregate 中，操作繁琐。
