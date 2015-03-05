---
layout: post
title:  "Openstack Heat : AutoScaling"
date:   2014-11-13 17:23:00
categories: OpenStack
---
本节来尝试一下heat实现的autoscaling集群

1.  环境要求  
    一个正常工作的openstack环境，　带有heat支持;  
    [metadata-server][1], 即http://169.254.169.254是正常工作的;  
    打算使用的镜像模板中安装了[heat-cfntools][2];  
    与普通的stack相比，一个autoscaling的stack需要有neutron提供loadbalance-as-a-service,　如何安装lbaas, 请参考[teststack][3]或者[手动安装][4], 后者是基于ubuntu的，在centos上安装需要自行适配;

2.  模板介绍  
    模板文件[autoscaling.yaml][5]和[lb_server.yaml][6]

autoscaling.yaml, 参数就不说了，大同小异

资源:

![autoscaling][7] 在ceilometer中建立两个alarm, 监控[cpu_util][8]; 在heat中建立两个scalingpolicy, 挂给ceilometer中的alarm; 当alarm被触发的时候, 就请求scalingpolicy提供的url; scalingpolicy执行相应的动作; 此例web_server_scaleup_policy在web_server_group中增加一个lb_server.yaml定义的资源; lb_server.yaml中定义的两个资源的组合, 一个是虚拟机实例, 另一个是往neutron中已有的loadbalancer中加一个member, 这样每当一个新的虚拟机实例被新加, 相应的loadbalancer也会将流量分流过来;

        resources:
        db:
            type: OS::Nova::Server
        web_server_group:
            type: OS::Heat::AutoScalingGroup
            properties:
            min_size: 1
            max_size: 3
            resource:
                type: lb_server.yaml
                properties:
                flavor: {get_param: flavor}
                image: {get_param: image}
                key_name: {get_param: key}
                pool_id: {get_resource: pool}
                metadata: {"metering.stack": {get_param: "OS::stack_id"}}
                user_data: ...
                ... ...
    
        web_server_scaleup_policy:
            type: OS::Heat::ScalingPolicy
            properties:
            adjustment_type: change_in_capacity
            auto_scaling_group_id: {get_resource: web_server_group}
            cooldown: 60
            scaling_adjustment: 1
        web_server_scaledown_policy:
            ... ...
        cpu_alarm_high:
            type: OS::Ceilometer::Alarm
            properties:
            # enabled: true/false
            description: Scale-up if the average CPU > 50% for 1 minute
            meter_name: cpu_util
            statistic: avg
            # lower peiod to see the effect.
            period: 60
            evaluation_periods: 1
            threshold: 50
            alarm_actions:
                - {get_attr: [web_server_scaleup_policy, alarm_url]}
            matching_metadata: {'metadata.user_metadata.stack': {get_param: "OS::stack_id"}}
            comparison_operator: gt
        cpu_alarm_low:
            ... ...
    

参数很多, 使用[environment][9]文件auto_env.yaml传递进去:

        parameters:
            image: fedora-heat-cfntools
            key: test
            flavor: m1.test
            database_flavor: m1.test
            subnet_id: 64025c10-2565-4d7a-8494-91e817fd3eb1
            database_name: wordpress
            database_user: wordpress
            external_network_id: b5b1a441-8b7c-4181-bcc2-08b899ae0132
    

注: [lbaas][10]

1.  测试 为了产生明显的效果, 需要在下面文件中更改ceilometer收集信息的频率 /etc/ceilometer/pipeline.yaml, 默认为600秒, 都改为30秒，　alarm中的period值的一半. 重启ceilometer.  
    也可以通过手动POST scalingpolicy的URL来立即看到效果:
    
        curl -X POST {scalingpolicy url}
        
    
    建立stack:
    
        heat stack-create autoscaling --template-file=autoscaling.yaml -e auto_env.yaml
        heat stack-list
        heat event-list autoscaling
        heat stack-show autoscaling
        
        nova list
        
        +--------------------------------------+-------------------------------------------------------+--------+------------+-------------+----------------------+
        | ID                                   | Name                                                  | Status | Task State | Power State | Networks             |
        +--------------------------------------+-------------------------------------------------------+--------+------------+-------------+----------------------+
        | b1abfe66-b6ee-48c0-8400-ef86229b23a5 | au-xrnt-vjv22xrgk7ki-6cbmnlhwuv7p-server-5a26kap5eaqd | ACTIVE | -          | Running     | demo-net=10.10.11.77 |
        | 1c8397e8-ddbd-47cd-bc16-309cbf12f322 | au-xrnt-ylgiwongp27b-m7vd2oqdebxj-server-57jguydvhbvf | ACTIVE | -          | Running     | demo-net=10.10.11.78 |
        | 1f79210b-3fcf-416a-a023-02a5a7800c35 | autoscaling-db-ej5yfc6mom7d                           | ACTIVE | -          | Running     | demo-net=10.10.11.76 |
        +--------------------------------------+-------------------------------------------------------+--------+------------+-------------+----------------------+
        
        neutron lb-member-list
        +--------------------------------------+-------------+---------------+----------------+--------+
        | id                                   | address     | protocol_port | admin_state_up | status |
        +--------------------------------------+-------------+---------------+----------------+--------+
        | 23e06e6f-b573-4a91-9a7c-690a422ddcff | 10.10.11.77 |            80 | True           | ACTIVE |
        +--------------------------------------+-------------+---------------+----------------+--------+
        
    
    下面进入虚拟机实例中制造负载
    
        ip netns exec qrouter-062a1052-3d55-4866-8555-c94be72c63a2 ssh fedora@10.10.11.77
        cat /dev/urandom > /dev/null &
        
        watch ceilometer alarm-list
        
        +--------------------------------------+-----------------------------------------+-------+---------+------------+--------------------------------+------------------+
        | Alarm ID                             | Name                                    | State | Enabled | Continuous | Alarm condition                | Time constraints |
        +--------------------------------------+-----------------------------------------+-------+---------+------------+--------------------------------+------------------+
        | 14d84ae5-ce2d-4130-81f3-19f9ab7e9553 | autoscaling-cpu_alarm_high-nc4qxili2fov | alarm | True    | False      | cpu_util > 50.0 during 1 x 60s | None             |
        | d89e8b6b-24c6-4eb3-b045-24625dfe8c01 | autoscaling-cpu_alarm_low-mrw5cw7autjt  | ok    | True    | False      | cpu_util < 15.0 during 1 x 60s | None             |
        +--------------------------------------+-----------------------------------------+-------+---------+------------+--------------------------------+------------------+
        
    
    当看到State为ok的时候, 意味着alarm被触发, 这时候通过nova list可以看到web_server_group中又新加了lb_server资源, 一个新实例, 并且loadbalancer中members也有新加
    
        neutron lb-pool-list
        +--------------------------------------+-------------------------------+----------+-------------+----------+----------------+--------+
        | id                                   | name                          | provider | lb_method   | protocol | admin_state_up | status |
        +--------------------------------------+-------------------------------+----------+-------------+----------+----------------+--------+
        | ae0c0d5f-951d-46ca-a09a-86e25f609638 | autoscaling-pool-ikgefqduak4h | haproxy  | ROUND_ROBIN | HTTP     | True           | ACTIVE |
        +--------------------------------------+-------------------------------+----------+-------------+----------+----------------+--------+
        neutron lb-member-list
        
        +--------------------------------------+-------------+---------------+----------------+--------+
        | id                                   | address     | protocol_port | admin_state_up | status |
        +--------------------------------------+-------------+---------------+----------------+--------+
        | 23e06e6f-b573-4a91-9a7c-690a422ddcff | 10.10.11.77 |            80 | True           | ACTIVE |
        | 741c532b-6d9c-4170-9355-aa07916ef953 | 10.10.11.78 |            80 | True           | ACTIVE |
        | f11b3582-9303-4272-a3e8-584a20bb4467 | 10.10.11.79 |            80 | True           | ACTIVE |
        +--------------------------------------+-------------+---------------+----------------+--------+
        
        neutron lb-pool-show ae0c0d5f-951d-46ca-a09a-86e25f609638                                          
        +------------------------+--------------------------------------------------------------------------------------------------------+
        | Field                  | Value                                                                                                  |
        +------------------------+--------------------------------------------------------------------------------------------------------+
        | admin_state_up         | True                                                                                                   |
        | description            |                                                                                                        |
        | health_monitors        | df31b0f8-8a6e-4193-ae79-e2e3a651c682                                                                   |
        | health_monitors_status | {"monitor_id": "df31b0f8-8a6e-4193-ae79-e2e3a651c682", "status": "ACTIVE", "status_description": null} |
        | id                     | ae0c0d5f-951d-46ca-a09a-86e25f609638                                                                   |
        | lb_method              | ROUND_ROBIN                                                                                            |
        | members                | 23e06e6f-b573-4a91-9a7c-690a422ddcff                                                                   |
        |                        | 741c532b-6d9c-4170-9355-aa07916ef953                                                                   |
        |                        | f11b3582-9303-4272-a3e8-584a20bb4467                                                                   |
        | name                   | autoscaling-pool-ikgefqduak4h                                                                          |
        | protocol               | HTTP                                                                                                   |
        | provider               | haproxy                                                                                                |
        | status                 | ACTIVE                                                                                                 |
        | status_description     |                                                                                                        |
        | subnet_id              | 64025c10-2565-4d7a-8494-91e817fd3eb1                                                                   |
        | tenant_id              | c9d81f3d3055411583507f2bf6b81d11                                                                       |
        | vip_id                 | da3b764d-5fc7-41ac-acc1-424229f6ad63                                                                   |
        +------------------------+--------------------------------------------------------------------------------------------------------+

 [1]: http://docs.openstack.org/icehouse/install-guide/install/yum/content/neutron-ml2-network-node.html
 [2]: /openstack/2014/10/15/openstack-heat-installation.html
 [3]: https://github.com/eayunstack/teststack
 [4]: http://docs.openstack.org/admin-guide-cloud/content/install_neutron-lbaas-agent.html
 [5]: https://github.com/openstack/heat-templates/blob/master/hot/autoscaling.yaml
 [6]: https://github.com/openstack/heat-templates/blob/master/hot/lb_server.yaml
 [7]: http://blog.apporc.org/wp-content/uploads/2014/11/autoscaling.png
 [8]: http://docs.openstack.org/developer/ceilometer/measurements.html
 [9]: http://docs.openstack.org/developer/heat/template_guide/environment.html#environments
 [10]: http://docs.openstack.org/api/openstack-network/2.0/content/lbaas_ext.html
