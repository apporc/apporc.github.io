## Heat支持的模版格式

*   HOT
*   CloudFormation

heat保持了对[AWS CloudFormation][1]模板的兼容, 同时自身也有一套native heat template format, 这套自身的模板格式正如官网所说的功能还有欠缺，仍在开发中.

## HOT介绍

HOT模板以yaml文件的形式存在, 一个HOT模板定义了一个叫作stack的应用提供出来

#### 模板结构

    heat_template_version: 2013-05-23
    
    description:
    # a description of the template
    
    parameter_groups:
    # a declaration of input parameter groups and order
    
    parameters:
    # declaration of input parameters
    
    resources:
    # declaration of template resources
    
    outputs:
    # declaration of output parameters
    

*   heat_template_version: 指定本yaml文档采用的HOT模板格式版本号, icehouse目前支持的版本号可以通过如下命令查询
    
        [root@openstack-network ~]# ip netns exec qdhcp-9aa4ebb9-4e93-416e-933f-fec4ce2ac1b3 curl http://169.254.169.254
        1.0
        2007-01-19
        2007-03-01
        2007-08-29
        2007-10-10
        2007-12-15
        2008-02-01
        2008-09-01
        2009-04-04
        

*   description: 对模板内容的描述性文字, 可以省略

*   parameter_groups: 将模板参数分组， 允许模板参数的分组传入, 可以省略

*   parameters: 指定本模板接受的参数， 定义参数名字类型等信息， 可省略

*   resources: 定义本模板所包含的资源， 资源至少有一个， 否则模板无意义

*   outputs: 当本模板被应用完成时， 反馈给用户的输出， 一般为用户期待的结果

#### Parameter Groups Section

指定参数的分组， 以及参数传递的顺序 每个组包含一个参数的列表， 列表中中的参数名对应于Parameters Section中的一个参数定义， 列表中的参数名不可重复

    parameter_groups:
    - label: <human-readable label of parameter group>
      description: <description of the parameter group>
      parameters:
      - <param name>
      - <param name>
    

*   label: 本参数组的标签

*   description: 描述

*   parameters: 本组中的参数列表

*   param name: 与Parameters Section中参数相对应的参数名称

#### Parameters Section

定义本模板接受的参数， 这些参数用于本模板部署的资源中. 每个参数的信息在一个独立的参数块中定义

    parameters:
    <param name>:
        type: <string | number | json | comma_delimited_list | boolean>
        label: <human-readable name of the parameter>
        description: <description of the parameter>
        default: <default value for parameter>
        hidden: <true | false>
        constraints:
        <parameter constraints>
    

*   param name: 参数名称

*   type: 标识参数的类型， 目前支持的类型有string, number, comma_delimited_list, json, or boolean.

*   label: 参数标签， 相比于param name的区别是这个为human-readable.

*   description: 描述性文字

*   default: 基于此模板的部署过程中， 对于应传而未传的参数采用这个属性指定的默认值

*   hidden: 在打印模板信息和日志的时候， 是否要隐藏这个参数所包含的信息， 例如密码等参数就需要隐藏， 默认为false

*   constraints: 对于参数值的限定， 如果用户提供了不符合限定的数值， 将会抛错. 详见Parameters Constraints.

**预定义参数:**

heat默认自定义了两个参数，分别为stack’s name 和identifier 它们名字分别为OS::stack_name和OS::stack_id

#### Parameter Constraints

在参数定义块中定义参数限定块， 格式如下:

    constraints:
    - <constraint type>: <constraint definition>
        description: <constraint description>
    

*   constraint type: 限定类型

*   constraint definition: 限定的具体内容

*   constraint description: 对限定所作的文字描述， 当用户提供的值不符合该限定的时候， 会将该文字描述提醒给用户.

**Constraint Types:**

*   length: 限定string类型的最大和最小长度， min和max必须提供至少一个.  
    格式:
    
        length: { min: <lower limit>, max: <upper limit> }
        

*   range: 限定number类型的最大值和最小值, min和max必须提供至少一个， 边界是被包含的.  
    格式:
    
        range: { min: <lower limit>, max: <upper limit> }
        

*   allowed_values: 限定number和string类型， 指定一系列可接受的值, 用户提供的值必须为这一系列值中的一个.  
    格式:
    
        allowed_values: [ <value>, <value>, ... ]
        
    
    或者
    
        allowed_values:
        - <value>
        - <value>
        - ...
        

*   allowed_pattern: 一个限定string类型的正则表达式  
    格式:
    
        allowed_pattern: <regular expression>
        

*   custom_constraint: 自定义限定类型, 通常用于检测相应资源是否真实存在. 自定义限定类型的定义在插件中实现，插件需要注册到heat engine. **插件的介绍, 待补充**  
    格式:
    
        custom_constraint: <name>
        

#### Resources Section

定义一个stack的组成资源, 每一个资源作为一个独立的区块, 格式:

    resources:
    <resource ID>:
        type: <resource type>
        properties:
        <property name>: <property value>
        metadata:
        <resource specific metadata>
        depends_on: <resource ID or list of ID>
        update_policy: <update policy>
        deletion_policy: <deletion policy>
    

*   resource ID resource ID是资源的唯一标识，在一个Resource Section中必须唯一
*   type 资源的类型, 如OS::Nova::Server.
*   properties 资源属性所组成的列表， 每个属性可以硬编码的， 也可以通过Intrinsic Functions获得, 可省略
*   metadata 特定资源类型所需的特定的属性， 一般用于一些高级功能， 提供update_policy,deletiong_policy一些必要的选项, 可省略
*   depends_on 资源的依赖资源, 可省略
*   update_policy: 以字典的形式定义此资源的更新策略, 具体的策略因资源类型不同而不同, 可省略
*   deletion_policy: 定义此资源的删除策略(Delete, Retain或Snapshot), 具体的策略因资源类型不同而不同, 可省略

**Resource Dependencies**

以资源resource ID来指定资源的依赖， 单一依赖直接指定， 多个依赖通过列表的形式指定

示例:

    resources:
    server1:
        type: OS::Nova::Server
        depends_on: [server2, server3]
    
    server2:
        type: OS::Nova::Server
    
    server3:
        type: OS::Nova::Server
    

**特定的资源类型将可以拥有此列表之外特定的属性**

#### Outputs Section

在一个单独的区块中定义需要返回给用户的参数信息， 如部署的实例的地址或者实例中部署的web服务的链接等.  
格式:

    outputs:
    <parameter name>:
        description: <description>
        value: <parameter value>
    

*   parameter name: 参数名称， 在Outputs Section中唯一

*   description: 参数的描述

*   parameter value: 参数的实际值, 通常使用Intrinsic Functions获取自特定资源.

示例:

    outputs:
    instance_ip:
        description: ip address of the deployed compute instance
        value: { get_attr: [my_instance, first_address] }
    

#### Intrinsic Functions

Heat提供了一系列在模板内部使用的函数， 完成一些特定任务， 例如获取一个实例的ip地址， 获取一个资源的属性等.

*   get_attr: 在运行时获取某一资源的某一属性, 对于属性为list/map等复杂结构的情况， 可以在列表中依次指定index值.  
    定义:
    
        get_attr:
        - <resource ID>
        - <attribute name>
        - <key/index 1> (optional)
        - <key/index 2> (optional)
        - ...
        

示例:

        resources:
        my_instance:
            type: OS::Nova::Server
            # ...
    
        outputs:
        instance_ip:
            description: IP address of the deployed compute instance
            value: { get_attr: [my_instance, first_address] }
        instance_private_ip:
            description: Private IP address of the deployed compute instance
            value: { get_attr: [my_instance, networks, private, 0] }
    

对于一个networks如下的实例:

    {"public": ["2001:0db8:0000:0000:0000:ff00:0042:8329", "1.2.3.4"],
    "private": ["10.0.0.1"]}
    

get_attr返回的值为"10.0.0.1"

*   get_file: 指定本地文件相对路径或url， 在模板部署时将文件内容自动读取进来替换在相应位置.  
    定义:
    
        get_file: <content key>
        
    
    示例:
    
        resources:
        my_instance:
            type: OS::Nova::Server
            properties:
            # general properties ...
            user_data:
                get_file: my_instance_user_data.sh
        my_other_instance:
            type: OS::Nova::Server
            properties:
            # general properties ...
            user_data:
                get_file: http://example.com/my_other_instance_user_data.sh
        

*   get_param: 运行时获取一个Parameters Section定义的输入参数值.  
    如果参数值为一个list/map等复杂结构， 可以通过列表指定一系列index值. 定义:
    
        get_param:
        - <parameter name>
        - <key/index 1> (optional)
        - <key/index 2> (optional)
        - ...
        
    
    示例:
    
        parameters:
        instance_type:
            type: string
            label: Instance Type
            description: Instance type to be used.
        server_data:
            type: json
        
        resources:
        my_instance:
            type: OS::Nova::Server
            properties:
            flavor: { get_param: instance_type}
            metadata: { get_param: [ server_data, metadata ] }
            key_name: { get_param: [ server_data, keys, 0 ] }
        

对于一个输入参数:

    {"instance_type": "m1.tiny",
    {"server_data": {"metadata": {"foo": "bar"},
                     "keys": ["a_key","other_key"]}}}
    

metadata将为{“foo”: “bar”}, key_name将为"a_key".

*   get_resource: 获取一个资源的reference ID.  
    例如， 对于一个floating ip资源， 将获取到它的ip地址.  
    格式:
    
        get_resource: <resource ID>
        

*   list_join: 以特定分隔符将列表转换为字符串. 类似于python中的 ','.join()的用法.  
    格式:
    
        list_join: [', ', ['one', 'two', 'and three']]
        
    
    返回“one, two, and three”

*   resource_facade: provider template从parent template中获取特定数据. 目前没有见到示例.  
    格式:
    
        resource_facade: <data type>
        
    
    date type为metadata, deletion_policy或者update_policy.

*   str_replace: 提供一个字符串或者字符串模板， 再提供相应参数， 可以实现将字符串中的相应占位符替换为相应参数.  
    格式:
    
        str_replace:
            template: <template string>
            params: <parameter mappings>
        
    
    示例:
    
    *   替换模板文件中的占位符
        
            resources:
            my_instance:
                type: OS::Nova::Server
                # general metadata and properties ...
            
            outputs:
            Login_URL:
                description: The URL to log into the deployed application
                value:
                str_replace:
                    template: http://host/MyApplication
                    params:
                    host: { get_attr: [ my_instance, first_address ] }
            
    
    *   替换字符串中的占位符
        
            parameters:
            DBRootPassword:
                type: string
                label: Database Password
                description: Root password for MySQL
                hidden: true
            
            resources:
            my_instance:
                type: OS::Nova::Server
                properties:
                # general properties ...
                user_data:
                    str_replace:
                    template: |
                        #!/bin/bash
                        echo "Hello world"
                        echo "Setting MySQL root password"
                        mysqladmin -u root password $db_rootpassword
                        # do more things ...
                    params:
                        $db_rootpassword: { get_param: DBRootPassword }
            

## CFN介绍

参考: [http://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/Welcome.html?r=7078][2]

## 示例模板

heat项目有一些示例的模板， 作为单独的一个repo存在:<https://github.com/openstack/heat-templates>

 [1]: http://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/Welcome.html?r=7078
 [2]: http://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/Welcome.html?r=7078]
