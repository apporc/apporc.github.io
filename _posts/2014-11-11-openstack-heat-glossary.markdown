---
layout: post
title:  "Openstack Heat : Glossary"
date:   2014-11-11 19:30:00
categories: OpenStack
---
*   API server  
    heat的Rest API服务
*   CFN  
    “AWS CloudFormation” 的简写
*   Constraint  
    模板中对于参数合法性的限定
*   Dependency  
    某一资源的存在是基于另外一些资源的存在，　即依赖关系. 存在heat内部暗含的以及用户显式定义的.
*   Environment  
    预定义一些参数以及资源类型等，在创建stack的过程上覆盖掉默认值
*   Heat Orchestration Template  
    heat的原生模板机制, 以YAML格式存在，不兼容CloudFormation的模板.
*   HOT  
    “Heat Orchestration Template”的简写
*   Input parameters  
    heat部署stack过程中接受来自用户输入的以及Environment里提供的参数
*   Metadata  
    可能是Resource Metadata, Nova Instance metadata, or the Metadata service.
*   Metadata service  
    nova-compute的一个服务, 实例可以从这个服务获取user-data等信息, 用于cloud-init,[cfn-init][1]等
*   Multi-region  
    heat的一个功能，支持在多个openstack部署之上统筹编排[Multi-region][2]
*   Nested resource  
    nested stack中的资源
*   Nested stack  
    一个模板中使用URL来引用了另一个stack的模板，　后者就是一个nested stack.
*   Nova Instance metadata  
    与实例相关的键值对, 疑似kwargs的作用
*   OpenStack  
    ...
*   Orchestrate  
    组织和领导openstack中特定场景中的特定的资源(元素?)以达成某一效果 Arrange or direct the elements of a situation to produce a desired effect.
*   Outputs  
    模板中一块信息，　定义本stack定义之后会反馈给用户的一些输出
*   Parameters  
    模板最前面的一个信息块，　用以定义创建和更新stack的时候接受的一些参数
*   Provider resource  
    一类资源，　资源的定义是由另外的模板(Provider template)完成的, 也就是nested stack. 本资源的property作为Provider template的输入参数
*   Provider template  
    一个用户定义的模板，　赋予用户以nested stack的形式自定义资源的能力, 其输出作为父stack的attributes
*   Resource  
    heat管理的资源，　资源有多种提供源，　见heat/engine/resources, 可以自己写resource plugin, 也可以使用nested stack作为自定义资源类型
*   Resource attribute  
    资源的属性
*   Resource group  
    资源组，　是一种特殊资源，　其打包了一堆常规资源
*   Resource Metadata  
    资源的一种属性, 其中可以填写CFN风格模板语言的metadata
*   Resource plugin  
    heat/engine/resources下面的一个python源文件, 定义了一种资源类型
*   Resource property  
    资源的属性, property是模板语言中定义一个资源具有的属性, 一个stack的networks属性, 在这个stack实例化之后，　这个networks的值{"public": ["2001:0db8:0000:0000:0000:ff00:0042:8329", "1.2.3.4"], "private": ["10.0.0.1"]}就是attributes. [difference between property and attribute][3]
*   Resource provider  
    特定资源的提供者, 一个Resource Plugin或者Provider template.
*   Stack  
    在一个模板中定义的资源集合生成的实例
*   Stack resource  
    一个资源, 这个资源本身是另一个stack. 由Provider resource提供.
*   Template  
    模板，　用于详细得描述一个stack的所有细节
*   Template resource  
    同Provider resource.
*   User data  
    资源的一种属性, 其中的信息将被传递给虚拟机实例的cloud-init，以在实例启动时配置实例
*   Wait condition  
    一种资源类型, 使得实例在可以与heat-engine进行交互, 以完成一些特定任务. 示例为:实例中正在进行配置，stack的创建被暂停.　避免stack说建完了, 但实际上实例中在配置的webserver实际并未启动

Reference: <http://docs.openstack.org/developer/heat/glossary.html>

 [1]: https://wiki.openstack.org/wiki/Heat/HA
 [2]: https://wiki.openstack.org/wiki/Heat/Blueprints/Multi_Region_Support_for_Heat
 [3]: http://english.stackexchange.com/questions/28085/what-is-the-difference-between-attribute-and-property
