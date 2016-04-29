#使用 OpenStack AutoScaling 实现可伸缩集群


## 背景

OpenStack 的 Heat 是 H 版以后新加入的一个组件，旨在创建一个机制来方便地管理
OpenStack 中的所有资源，资源包括虚拟机、卷、镜像等等。用户可以通过 Heat 来
对这些资源进行整合与分组，添加或删除，按批量的管理。

而 AutoScaling，即可伸缩集群，只是 Heat 所提供的一个子功能。通过定义一个可伸
缩集群，将虚拟机、卷以及网络等资源组合在集群中并能够响应一些事件来自动地调整
集群的大小。

## 功能

AutoScaling 是一个刚刚起步的功能，功能尚未完整。

已实现的功能：

1.  用户可以通过 Heat 模板来创建可伸缩集群。
2.  可伸缩的功能不仅仅应用在虚拟机上，对其他的资源也可伸缩。
3.  创建自动伸缩集群的同时，提供给用户其他资源，比如负载均衡，集群管理，虚机配
    置等。

待实现的功能：
1.  用户可以通过 Heat 的 API 直接创建可伸缩集群。(开发进行中)
2.  在对集群作伸缩时，能够指定是同类资源中的哪一个，比如一个集群有三个虚拟机，
    在删除时需要能指定要删哪一个，而不是随机选择。
3.  针对减缩集群，应允许用户指定某种通用的策略，比如优先删除最新添加的资源。
4.  提供一个回调链接，当要进行减缩集群操作的时候，允许虚拟机内通过链接来确认操
    作。例如，对于数据库集群等，在删除节点之前应给予其机会作数据的同步。
5.  提供一个回调链接，当要进行减缩集群操作的时候，允许虚拟机在认定自己为应缩减
    对象之后，调用链接来自我删除。

## 使用

### 使用 API

AutoScaling API 正在开发中，API 设计可以参考
http://docs.heatautoscale.apiary.io/

### 使用模板

在 Heat 中使用模板的步骤是：1. 编写一个模板文件，2.把文件通过如下命令传递给
Heat：`heat stack-create <name> --template-file=<模板文件>`

Heat 把每个模板在 OpenStack 环境中所对应的对象称作 stack，其全套的操作文档是以
stack 开头的各个子命令：`heat stack-*`。

Heat 的命令用法本文不作描述，可以使用 `heat help` 查询。

在使用 AutoScaling 之前，需要先了解 Heat 模板的写法，因为毕竟 AutoScaling 在模
板中仅作为几种资源存在。

Heat 模板格式介绍参考：[模板格式][1] 或英文原版 [Template Specification][3]
Heat 模板中已支持的资源类型汇总：[resource types][2]

下面说一下 AutoScaling 模板具体涉及到的三个资源类型：ScalingGroup、ScalingPolicy
、WebHook

|名称|类型|属性|
|----|----|----|
|ScalingGroup|OS::Heat::AutoScalingGroup|max_size: 可伸缩资源数量的最大值<br>
min_size：可伸缩资源数量的最小值<br>
resource：定义可伸缩资源组的子资源<br>
cooldown：以秒为单位的伸缩动作冷却时间<br>
desired_capacity：可伸缩资源的初始数量<br>
current_size：获取可伸缩集群当前的资源数量<br>|
|ScalingPolicy|OS::Heat::ScalingPolicy|adjustment_type：对可伸缩资源组所作的
调整动作的类型，包括三种选择。change_in_capacity：变更一定的步长，
exact_capacity：变更为某特定的数量，percent_change_in_capacity：按百分比进行变更。<br>
auto_scaling_group_id：所要变更的可伸缩资源组的 id。<br>
scaling_adjustment：变更的量。<br>
cooldown：变更的冷却时间。<br>
min_adjustment_step：最小变更步长。<br>
alarm_url：一个变更策略资源会暴露一个 url，通过对 url 的访问即可实现其所定义的变更。
url 背后是 AutoScaling 实现的 WebHook。|

1.  ScalingGroup 定义可伸缩集群的资源，这些资源本身是 Heat 模板所支持的资源类型
    。另外，ScalingGroup 也定义可伸缩资源数量的上下限。

    示例：

        # server_group 为资源名称
        server_group:
            # type 指定资源的类型
            type: OS::Heat::AutoScalingGroup
            # properties 指定资源的各项属性，每种资源类型对应不同的资源属性。
            properties:
                # min_size 与 max_size 指定可伸缩资源的数量上下限，示例中指定
                # 最少与最多的虚拟机实例数目分别为 1 和 3，则实际的集群在这个
                # 范围内进行伸缩。
                min_size: 1
                max_size: 3
                # 在 resource 属性中定义本 AutoScalingGroup 中的子资源。
                resource:
                    # 在此指定资源类型为 OS::Nova::Server 类型，即虚拟机实例。
                    # 此子资源的属性依照 OS::Nova::Server 类型的要求进行指定。
                    type: OS::Nova::Server
                        # 以下是创建虚拟机所需要指定的一些参数。
                        properties:
                        flavor: {get_param: database_flavor}
                        image: {get_param: image}
                        key_name: {get_param: key}
                        networks: [{"network":{get_param: network}}]

2.  ScalingPolicy 用来设定一个 ScalingGroup 伸缩的策略，包括是增还是减，以及每
    次伸缩的跨度和伸缩的频度控制等。

    以下为对上述示例资源 server_group 的两个伸缩策略，分别为增加与缩减。

        # 伸缩策略也是一种资源，需要有资源名称
        scaleup_policy:
            # 伸缩资源的类型为 OS::Heat::ScalingPolicy
            type: OS::Heat::ScalingPolicy
            properties:
                # 以下伸缩资源需要定义的各属性
                # 指定变更动作为按数量变更
                adjustment_type: change_in_capacity
                # 指定本伸缩策略所针对的可伸缩资源组，这儿使用了内部函数
                # get_resource，更多函数可以参考 Heat 模板格式
                auto_scaling_group_id: {get_resource: server_group}
                cooldown: 60
                # 由于变更动作为按数量变更，这儿的数字 1 指定每变更的幅度为一个
                # 虚拟机。
                scaling_adjustment: 1
        # 以下为一个相应的缩减策略，与上面的增加策略基本相同，只是指定变更幅度
        # 为 -1。
        scaledown_policy:
            type: OS::Heat::ScalingPolicy
            properties:
                adjustment_type: change_in_capacity
                auto_scaling_group_id: {get_resource: server_group}
                cooldown: 60
            scaling_adjustment: -1

3.  WebHook 一个 ScalingPolicy 对应一个 WebHook，以 url 的形式显现，访问 url 即
    调用 WebHook。

    获取上述第二步所建立的两个伸缩策略的 url。

        scale_up_url:
            description: >
            This URL is the webhook to scale up the autoscaling group.  You
            can invoke the scale-up operation by doing an HTTP POST to this
            URL; no body nor extra headers are needed.
            value: {get_attr: [scaleup_policy, alarm_url]}
        scale_dn_url:
            description: >
            This URL is the webhook to scale down the autoscaling group.
            You can invoke the scale-down operation by doing an HTTP POST to
            this URL; no body nor extra headers are needed.
            value: {get_attr: [scaledown_policy, alarm_url]}

    参见注释，只需要对相应 url 进行 HTTP POST 访问即可触发相应的伸缩动作。
    也就是说，管理员可以在任意时刻通过手动 POST WebHook 的 url 来实现伸缩动作。
    当然，如果要实现自动的伸缩，就需要建立另外的自动化机制来调用 WebHook。
    这个自动化的机制即是 Ceilometer 的警报 (alarm)，Ceilometer 可以通过定时收集
    集群状态信息，并在达到要求时发起对 WebHook 的调用。

### 测试示例

注：目前必须以管理员身份才能创建 AutoScaling 集群，因为创建 AutoScalingGroup 时
会需要创建用户。

#### 测试一：创建可手动伸缩的简单集群

参见[Heat 模板结构][1]，模板中主要的部分只有：parameters，resources 和 outputs
三个。针对同一个版本的 OpenStack部分，heat_template_version 可以保持不变。

以下我们按顺序编写每一部分，并解释。

*   heat_template_version:
    在 juno 版本暂时没有查询 heat 支持的模板格式列表的命令，不
    过已知其支持的版本列表为：2014-10-16 和 2013-05-23

        heat_template_version: 2014-10-16

*   parameters：
    参数列表，在使用 `heat stack-create` 时，可以指定参数值。

        parameters:
            image:
                type: string
                description: Image used for servers
            key_name:
                type: string
                description: SSH key to connect to the servers
            flavor:
                type: string
                description: flavor used by the web servers
            network:
                type: string
                description: network to use

    我们在参数列表中，定义了创建虚拟机时所使用的镜像名称(image)，密钥名称
    (key_name)，规格名(flavor)和网络名称(network)，这些都是创建虚拟机所需指定
    的基本信息。

*  resources：
    资源列表，一个模板中最核心的部分即是其所定义的资源。Heat 在收到这个模板之
    后，会将资源列表中的各个资源按要求创建。

        resources:
        # 本资源列表中，我们定义三个资源，分别为一个 AutoScalingGroup 和两个
        # ScalingPolicy 类型的资源。AutoScalingGroup 是我们的虚拟机集群，两
        # 个 ScalingPolicy 资源分别是增加和缩减集群中虚拟机实例数量的策略。
            server_group:
                type: OS::Heat::AutoScalingGroup
                properties:
                    min_size: 1
                    max_size: 3
                    resource:
                    # AutoScalingGroup 的子资源列表在此指定，对于本次测试来说，
                    # 组中唯一的资源即是虚拟机，其资源类型为 OS::Nova::Server。
                        type: OS::Nova::Server
                        properties:
                        # 传递 parameters 列表中的参数值至子资源。所用到的内部
                        # 函数为 get_param，更多内部函数，参见模板格式那一篇。

                            flavor: {get_param: flavor}
                            image: {get_param: image}
                            key_name: {get_param: key_name}
                            networks: [{"network": {get_param: network}}]
            # ScalingPolicy 的各属性含义上文已说，这里不再重复。
            scaleup_policy:
                type: OS::Heat::ScalingPolicy
                properties:
                    adjustment_type: change_in_capacity
                    auto_scaling_group_id: {get_resource: server_group}
                    cooldown: 60
                    scaling_adjustment: 1
            scaledown_policy:
                type: OS::Heat::ScalingPolicy
                properties:
                    adjustment_type: change_in_capacity
                    auto_scaling_group_id: {get_resource: server_group}
                    cooldown: 60
                    scaling_adjustment: -1

*   outputs：
    stack 创建成功之后反馈给用户的信息。
            
        outputs:
        # 輸出两个信息，分别对应上文两个 ScalingPolicy 所定义的两个 WebHook 的
        # 链接(url)。通过对 url 作 POST 可以触发伸缩动作。输出信息在 horizon
        # 界面上会显示在 Orchestration (编排) 标签下面的 Stacks 中新建的 stack
        # 信息中。
            scale_up_url:
                description: >
                    This URL is the webhook to scale up the autoscaling group.  You
                    can invoke the scale-up operation by doing an HTTP POST to this
                    URL; no body nor extra headers are needed.
                # 从 ScalingPolicy 获取其 WebHook url 的函数是内部函数 get_attr，更
                # 多内部函数，参见模板格式那一篇。
                value: {get_attr: [scaleup_policy, alarm_url]}
            scale_dn_url:
                description: >
                    This URL is the webhook to scale down the autoscaling group.
                    You can invoke the scale-down operation by doing an HTTP POST to
                    this URL; no body nor extra headers are needed.
                value: {get_attr: [scaledown_policy, alarm_url]}

至此，我们已编写完模板的各部分内容，将以上各部分组合起来，文件下载地址：[base.yaml][4]
在测试环境中发起测试，参数列表的值，根据情况请自行设定：

    #. apporc_openrc
    #heat stack-create --template-file base.yaml --parameters "image=TestVM;flavor=m1.micro;key_name=apporc;network=apporc_net" base

    +--------------------------------------+------------+--------------------+----------------------+
    | id                                   | stack_name | stack_status       | creation_time        |
    +--------------------------------------+------------+--------------------+----------------------+
    | 6f367e4e-7f22-4a4b-b979-5f4438fd8026 | base       | CREATE_IN_PROGRESS | 2015-09-18T07:16:51Z |
    +--------------------------------------+------------+--------------------+----------------------+

创建完成，可以通过使用 `nova list` 查看有一个新建成的虚拟机，名称很长，其中有
stack 名称的前缀以及可伸缩资源组的名称 server_group，即是新建的虚拟机。

注：目前 Heat 没有提供查询 stack 与其子资源对应关系的 API。

通过命令行或者 horizon 界面，查看反馈信息：

    #heat stack-show base

    ... ...
    |                      |     "output_value": "http://25.0.0.2:8000/v1/signal/arn%3Aopenstack%3Aheat%3A%3A578cc76cce794f15ac5b6417b9480e26%3Astacks%2Fbase%2F6f367e4e-7f22-4a4b-b979-5f4438fd802
    6%2Fresources%2Fscaledown_policy?Timestamp=2015-09-18T07%3A17%3A03Z&SignatureMethod=HmacSHA256&AWSAccessKeyId=7b06a374fcfb45ceab36933737ebca57&SignatureVersion=2&Signature=yAC3aSWy5UUyW3eccmq
    D%2FVJScJVlzJyPIykfTa7JMeY%3D",  |
    |                      |     "description": "This URL is the webhook to scale down the autoscaling group. You can invoke the scale-down operation by doing an HTTP POST to this URL; no body no
    r extra headers are needed.\n",                                                                                                                                                                
                                    |
    |                      |     "output_key": "scale_dn_url" 
    |                      |     "output_value": "http://25.0.0.2:8000/v1/signal/arn%3Aopenstack%3Aheat%3A%3A578cc76cce794f15ac5b6417b9480e26%3Astacks%2Fbase%2F6f367e4e-7f22-4a4b-b979-5f4438fd802
    6%2Fresources%2Fscaleup_policy?Timestamp=2015-09-18T07%3A17%3A03Z&SignatureMethod=HmacSHA256&AWSAccessKeyId=638f6f2ecb7d4827b58716cb31060b8b&SignatureVersion=2&Signature=NHusMRueBXe1LhLRiQPFY
    qu3AW1ziVneKBoUWrD1vuk%3D",      |
    |                      |     "description": "This URL is the webhook to scale up the autoscaling group.  You can invoke the scale-up operation by doing an HTTP POST to this URL; no body nor e
    xtra headers are needed.\n",                                                                                                                                                                   
                                    |
    |                      |     "output_key": "scale_up_url" 

也可以使用 `heat output-list base` 和 `heat output-show base scle_up_url` 的方
式来查询输出信息。

从中我们可以看到两个 WebHook 对应的 url。下面可以使用 curl 来访问 scale_up_url
和 sclae_dn_url 来看看效果。

    #curl -X POST 'http://25.0.0.2:8000/v1/signal/arn%3Aopenstack%3Aheat%3A%3A578cc76cce794f15ac5b6417b9480e26%3Astacks%2Fbase%2F6f367e4e-7f22-4a4b-b979-5f4438fd8026%2Fresources%2Fscaleup_policy?Timestamp=2015-09-18T07%3A17%3A03Z&SignatureMethod=HmacSHA256&AWSAccessKeyId=638f6f2ecb7d4827b58716cb31060b8b&SignatureVersion=2&Signature=NHusMRueBXe1LhLRiQPFYqu3AW1ziVneKBoUWrD1vuk%3D'
    #nova list
可以看到在 POST 之后，又有一个新虚拟机建立出来。

由于我们设置 server_group 的伸缩范围为 1-3 (参见模板中的 min_size 和 max_size)
，所以不能一开始就作缩减的，不会有效果。先增加之后再缩减来看效果即可。

下面回顾一下，上述过程有一个地方非常烦，就是 stack-create 命令行处，需要指定
很多参数，此次我们只指定了四个参数，假如以后有十几二十几个参数，岂不是非常烦
恼。幸好传参的过程是有简化的办法的，下面来看：

Heat 支持将参数集合在一个 yaml 文件中在进行 stack-create 时集体传入，参
见[Environment File][5]。
我们新建一个文件 [base_env.yaml][6]：

    parameters:
        image: TestVM
        key_name: apporc
        flavor: m1.micro
        network: apporc_net

然后命令就变成这个样子了：

    #heat stack-create --template-file base.yaml --environment-file base_env.yaml base

至此测试一完成，遗留的问题是我们所需要的当然是自动伸缩的集群，通过这样手动去
执行伸缩动作，显然不是我们的期望。下面再看测试二。

#### 测试二：

前方已提及，自动的去访问 WebHook 触发伸缩操作，需要用到 Ceilometer 的 Alarm。

详细的 Alarm 的使用方法，属于 Ceilometer 的功能，本文不详述，可以参考
[ceilometer command line][7] 和 [Telemetry Service Using][8]。

*   resources 更改：

        resources:
            server_group:
                type: OS::Heat::AutoScalingGroup
                properties:
                    min_size: 1
                    max_size: 3
                    resource:
                        type: OS::Nova::Server
                        properties:
                            flavor: {get_param: flavor}
                            image: {get_param: image}
                            key_name: {get_param: key_name}
                            # 添加一个 metadata，该 metadata 标识了虚拟机所属 stack 的 id
                            metadata: {"metering.stack": {get_param: "OS::stack_id"}}
                            networks: [{"network": {get_param: network}}]

*   添加 Alarm 资源：
    添加两个 OS::Ceilometer::Alarm 类型的资源，这两个资源的作用正好对应于两个
    ScalingPolicy，一个 Alarm 用于自动触发集群增加动作，一个自动触发缩减动作。
    关于 Ceilometer Alarm 资源属性的详细介绍，参见：[OS::Ceilometer::Alarm][8]

        cpu_alarm_high:
            type: OS::Ceilometer::Alarm
            properties:
                # 当集群中虚拟机实例的 CPU 一分钟内平均使用率大于 50%
                # 就进行增加动作，增加集群虚拟机实例。
                description: Scale-up if the average CPU > 50% for 1 minute
                # meter_name 指定检测样本为 cpu_util，也就是 cpu 使用率。
                meter_name: cpu_util
                # 计算数据的平均值。
                statistic: avg
                # 一轮数据采集的时间跨度。
                period: 60
                # 需要几轮数据
                evaluation_periods: 1
                # 阈值，是否触发一个 Alarm 需要两部分条件，threshold 跟下边的
                # comparison_operator，即与阈值的比较方法。
                threshold: 50
                # 阈值达到时，所触发的动作内容，此处动作内容为 WebHook 的链接。
                alarm_actions:
                    - {get_attr: [scaleup_policy, alarm_url]}
                # 只有那些匹配特定 metadata 特征的虚拟机，才是操作对象。
                matching_metadata: {'metadata.user_metadata.stack': {get_param: "OS::stack_id"}}
                # 与阈值的比较方法，gt 是大于，lt 是小于。
                comparison_operator: gt
        cpu_alarm_low:
            type: OS::Ceilometer::Alarm
            properties:
                description: Scale-down if the average CPU < 15% for 1 minute
                meter_name: cpu_util
                statistic: avg
                period: 60
                evaluation_periods: 1
                threshold: 15
                alarm_actions:
                    - {get_attr: [scaledown_policy, alarm_url]}
                matching_metadata: {'metadata.user_metadata.stack': {get_param: "OS::stack_id"}}
                comparison_operator: lt

以上就是测试二的更改，全部文件在[withalarm.yaml][10]。
发起创建：

    heat stack-create --template-file withalarm.yaml --environment-file base_env.yaml withalarm

通过 `ceilometer alarm-list` 查看新建的 alarm。

    # ceilometer alarm-list
    +--------------------------------------+---------------------------------------+-------------------+---------+------------+--------------------------------+------------------+
    | Alarm ID                             | Name                                  | State             | Enabled | Continuous | Alarm condition                | Time constraints |
    +--------------------------------------+---------------------------------------+-------------------+---------+------------+--------------------------------+------------------+
    | 2dd29a00-9b1f-4cfc-9417-44d6640872fc | withalarm-cpu_alarm_high-tb26eotmmb7v | insufficient data | True    | False      | cpu_util > 50.0 during 1 x 60s | None             |
    | 944b8bd3-6407-4a33-bcf5-1ba91a6ba738 | withalarm-cpu_alarm_low-gkmnp5naddky  | insufficient data | True    | False      | cpu_util < 15.0 during 1 x 60s | None             |
    +--------------------------------------+---------------------------------------+-------------------+---------+------------+--------------------------------+------------------+
可以看到两个 Alarm 的名称与状态等信息，示例中的 State 一栏为"insufficient data"
意为收集到的数据尚不足。Ceilometer 在 /etc/ceilometer/pipeline.yaml 中为所有的
数据采样定义了一个收集周期。测试中想要更快的看到效果，就要更改一下配置，将默认
的 600 秒缩小至 50 秒(根据我的理解，该值要小于我们上面定义的 Alarm 的评估周期)。
由于本次测试只用到了 cpu_util 一项数据，故测试中我只更改了 meter_source 和 
cpu_source 的 interval 值。
建议更改完，重启所有 ceilometer 服务(systemd 管理的，或者 pacemaker 管理的，
ceilometer-alarm-evaluator 服务目前可能有 bug，pacemaker 显示始终为 Stopped 
状态，不过不影响测试)。

注：一个较小的采集周期，可能会为 Ceilometer 带来巨大的压力，测试完成请调回常
规值。

当 ceilometer 收集到足够的数据，State 为 OK，如果 alarm 被触发，则 State 为
alarm。

下面我们需要去我们建立的虚拟机中制造负载。 虚拟机名为本次 stack 的名称
withalarm 的前两个字母作前缀，跟 server_group 的组合，示例中为：
`wi-erver_group-umbsyaoikttc-pi5wqwqfag7x-w4n3fubqk2jv`

连接中虚拟机，对于 N 核心的虚拟机，执行如下命令 N 次来制造负载：

    $cat /dev/urandom > /dev/null &

之后，坐等 alarm 被触发。当 alarm 被触发时， heat 可以通过
`heat event-list <stack name>` 来查询到触发动作。
示例中，过了一分钟， alarm 被准确地触发了：


    # heat event-list withalarm
    +------------------+--------------------------------------+---------------------------------------------------------------------------------------------------------------------------------+--------------------+----------------------+
    | resource_name    | id                                   | resource_status_reason                                                                                                          | resource_status    | event_time           |
    +------------------+--------------------------------------+---------------------------------------------------------------------------------------------------------------------------------+--------------------+----------------------+
    | scaleup_policy   | bf088f32-6d21-4299-a6e6-96d1efb2647c | alarm state changed from ok to alarm (Transition to alarm due to 1 samples outside threshold, most recent: 91.04)               | signal_COMPLETE    | 2015-09-21T02:56:17Z |

此时通过` nova list ` 应该可以看到新增加了一个虚拟机节点。测试通过。


注：如果需要重复触发 alarm_actions，需要设置 alarm 资源的 repeat_actions 为
true。 文档 [repeat_actions][11] 中说 repeat_actions 默认为 true，应该是写错了
，或者版本跟我们不一致。
以上我们成功创建了自动伸缩的集群，是能够伸缩，而且是自动的。然而测试离现实场景
还有一段距离。现实中我们肯定需要集群中的各节点是共同提供同一项服务的，而非
测试中这样各自独立。

下面的测试中，我们尝试将 autoscaling 集群创建为一个 httpd 的负载均衡集群。
并使

#### 测试三：加入负载均衡器

负载均衡器属于 Neutron 负载均衡服务的一个组件，关于负载均衡服务使用的方法，请先
阅读[lbaas][12]。基本上来说，对于负载均衡的几个元素，每个我们都在模板中定义其
对应的资源。

*   参数
    设置负载均衡器，需要用到几个新的参数。外网 id 与子网 id，负载均衡器对外使用
    一个外网地址，对内将网络连接分发至多个子网地址上。

        subnet_id:
            type: string
            description: subnet on which the load balancer will be located(UUID)
        external_network_id:
            type: string
            description: UUID of a Neutron external network

*   负载均衡器
    几个基本元素分别为：pool、member、vip 以及 healthmonitor，其所对应的资源
    定义示例如下，每种资源的各属性含义参考[resource types][2]。

        lb:
            # 首先定义一个 LoadBalancer 的资源，后续再定义其几个要素。
            type: OS::Neutron::LoadBalancer
            properties:
            # protocol_port 指定是针对 tcp 80 端口的负载均衡
            protocol_port: 80
            pool_id: {get_resource: pool}
        monitor:
            # HealthMonitor 用于检测各节点服务健康状况
            type: OS::Neutron::HealthMonitor
            properties:
            type: TCP
            delay: 5
            max_retries: 5
            timeout: 5
        pool:
            type: OS::Neutron::Pool
            properties:
            protocol: HTTP
            monitors: [{get_resource: monitor}]
            subnet_id: {get_param: subnet_id}
            # 负载均衡的方式，这里设为轮询
            lb_method: ROUND_ROBIN
            vip:
                # 虚拟 IP 对外侦听的服务端口
                protocol_port: 80
        lb_floating:
            # 定义一个浮动 IP 地址的资源来作为负载均衡器的虚拟 IP
            type: "OS::Neutron::FloatingIP"
            properties:
            floating_network_id: {get_param: external_network_id}
            port_id: {get_attr: [pool, vip, port_id]}

    除了上述已定义的 HealthMonitor，Pool 和 FloatingIP 之外，负载均衡器的第四个
    要素 PoolMember，在可伸缩集群背景下，应该是一个可伸缩资源。每有一个新的集群
    节点加入进来，就对应得要有一个新的 PoolMember 资源。显然 PoolMember 资源
    应该是属于 AutoScalingGroup 中的子资源，与 Nover::Server 并列。

    由于 AutoScalingGroup 中的 resource 属性中只能指定一种子资源类型，而非一系列
    子资源的列表，故而我们需要用到 Heat 模板的一个特性"自定义资源类型"。自定义
    资源类型是通过定义一个特定格式的 yaml 文件来达成的。其格式与定义一个 Heat
    模板的格式一致。

    略过参数列表，示例中我们定义了一个 [lb_server.yaml][13]：

        resources:
        server:
            type: OS::Nova::Server
            properties:
                flavor: {get_param: flavor}
                image: {get_param: image}
                key_name: {get_param: key_name}
                metadata: {get_param: metadata}
                networks: [{"network": {get_param: network}}]
        member:
            type: OS::Neutron::PoolMember
            properties:
                # 属性中将 LoadBalancer Pool 的 id 与新加节点 server 的第一个
                # ip 地址进行关联。
                pool_id: {get_param: pool_id}
                address: {get_attr: [server, first_address]}
                protocol_port: 80

    使用一个"自定义资源类型"的时候，只需将资源的 type 指定为相应 yaml 文件名，
    并传入所需参数即可，使用自定义资源类型之后， AutoScalingGroup 定义如下：

        server_group:
            type: OS::Heat::AutoScalingGroup
            properties:
            min_size: 1
            max_size: 3
            resource:
                type: lb_server.yaml
                properties:
                    flavor: {get_param: flavor}
                    image: {get_param: image}
                    key_name: {get_param: key_name}
                    pool_id: {get_resource: pool}
                    network: {get_param: network}
                    metadata: {"metering.stack": {get_param: "OS::stack_id"}}

*   userdata
    以上我们虽然配置了 tcp 80 端口的负载均衡器，然而集群中各节点尚未对 80 端口
    提供服务，是以我们需要在各节点上安装 httpd 服务。当然可以通过预先在模板中
    安装 httpd 并默认启动来达成目的，但配置集群的过程中难免会有一些需要运行时
    进行干预的配置项，所以我们在这里以 userdata 的方法来示例。

    OS::Nova::Server 资源类型接受 userdata 来在虚拟机初次启动时对虚拟机进行
    自动配置。为此我们在 lb_server.yaml 中添加 userdata 支持。

        resources:
        server:
            type: OS::Nova::Server
            properties:
            flavor: {get_param: flavor}
            image: {get_param: image}
            key_name: {get_param: key_name}
            metadata: {get_param: metadata}
            networks: [{"network": {get_param: network}}]
            # 接收 user_data 参数，将参数值传递给 OS::Nova::Server
            user_data: {get_param: user_data}
            # 指定接受的 user_data 参数值的类型。
            user_data_format: RAW

    并在 AutoScalingGroup 中将 user_data 值传递给 lb_server.yaml：

        resource:
            type: lb_server.yaml
            properties:
            flavor: {get_param: flavor}
            image: {get_param: image}
            key_name: {get_param: key_name}
            pool_id: {get_resource: pool}
            network: {get_param: network}
            metadata: {"metering.stack": {get_param: "OS::stack_id"}}
            user_data:
                str_replace:
                # 使用 str_replace 的格式，可以方便地编写 shell 脚本，来自动
                # 化的配置虚拟机，脚本内容放置在 template 关键字之后，脚本中
                # 可以使用一些变量，其值可以在 params 关键字中指定。
                template: |
                    #!/bin/bash -v
                    yum -y install httpd
                    systemctl enable httpd.service
                    systemctl start httpd.service
                    echo "$HOSTNAME" >  /usr/share/httpd/noindex/index.html
                    echo "$stack_id" >> /usr/share/httpd/noindex/index.html
                params:
                    # 这里传递了 stack_id 进 shell 脚本，作为变量传递的示例。
                    $stack_id: {get_param: "OS::stack_id"}

    加入负载均衡器之后的完整模板文件为[withlb.yaml][14]，参数列表文件
    为[lb_env.yaml][15]。创建 stack：

        heat stack-create --template-file withlb.yaml --environment-file lb_env.yaml withlb

    withlb.yaml 模板的 outputs 部分中加入了对负载均衡 httpd 访问地址的输出。可
    以直接访问相应地址。

        # heat output-show withlb website_url
        "http://25.0.1.28/"

    步骤同上，进入虚拟机制造负载，然后隔几秒访问一下 website_url，一开始网页内容
    中是集群第一个节点的名称，之后当新节点加入之后，网页内容有时为第一个节点的名
    称，有时为第二个节点的名称。就证明了自动伸缩的负载均衡集群已经搭建成功。

注：以上测试所用到的模板文件，我都放在[eayunstack heat-templates repo][16]中，
    对于之后我们作测试，可以都把模板整合在该 repo 之中。

 [1]: https://oa.eayun.cn/wiki/doku.php?id=eayunstack:heat:template_format
 [2]: http://docs.openstack.org/developer/heat/template_guide/openstack.html
 [3]: http://docs.openstack.org/developer/heat/template_guide/hot_spec.html
 [4]: https://raw.githubusercontent.com/eayunstack/heat-templates/master/base.yaml
 [5]: http://docs.openstack.org/developer/heat/template_guide/environment.html#environments
 [6]: https://raw.githubusercontent.com/eayunstack/heat-templates/master/base_env.yaml
 [7]: http://docs.openstack.org/cli-reference/content/ceilometerclient_commands.html
 [8]: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/6/html/Administration_Guide/sect-Using_the_Telemetry_Service.html
 [9]: http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Ceilometer::Alarm
 [10]: https://raw.githubusercontent.com/eayunstack/heat-templates/master/withalarm.yaml
 [11]: http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Ceilometer::Alarm-prop-repeat_actions
 [12]: https://oa.eayun.cn/wiki/doku.php?id=eayunstack:neutron:lbaas
 [13]: https://raw.githubusercontent.com/eayunstack/heat-templates/master/lb_server.yaml
 [14]: https://raw.githubusercontent.com/eayunstack/heat-templates/master/withlb.yaml
 [15]: https://raw.githubusercontent.com/eayunstack/heat-templates/master/lb_env.yaml
 [16]: https://github.com/eayunstack/heat-templates
