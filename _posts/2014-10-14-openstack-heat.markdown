---
layout: post
title:  "Openstack Heat"
date:   2014-10-14 17:25:00
categories: OpenStack
---
#### 1\. What is heat

*   About [CloudFormation][1]: AWS CloudFormation enables you to create and provision AWS infrastructure deployments predictably and repeatedly. It helps you leverage AWS products such as Amazon EC2, Amazon Elastic Block Store, Amazon SNS, Elastic Load Balancing, and Auto Scaling to build highly reliable, highly scalable, cost-effective applications without worrying about creating and configuring the underlying AWS infrastructure. AWS CloudFormation enables you to use a template file to create and delete a collection of resources together as a single unit (a stack).

*   About Orchestration: The mission of the OpenStack Orchestration program is to create a human- and machine-accessible service for managing the entire lifecycle of infrastructure and applications within OpenStack clouds.

*   About [Heat][2]: Heat is the main project in the OpenStack Orchestration program. Heat is a service to orchestrate multiple composite cloud applications using the AWS CloudFormation template format, through both an OpenStack-native ReST API and a CloudFormation-compatible API.
    
    From [The official list of OpenStack programs][3]:
    
        Orchestration:
        codename: Heat
        ptl: Zane Bitter (zaneb)
        url: https://wiki.openstack.org/wiki/Heat
        projects:
            - repo: openstack/heat
            incubated-since: grizzly
            integrated-since: havana
            - repo: openstack/python-heatclient
            - repo: openstack/heat-cfntools
            - repo: openstack/heat-specs
            - repo: openstack/heat-templates
            - repo: openstack-dev/heat-cfnclient
        

*   Purpose of Heat
    
    *   Heat provides a template based orchestration for describing a cloud application by executing appropriate OpenStack API calls to generate running cloud applications.
    *   The software integrates other core components of OpenStack into a one-file template system. The templates allow creation of most OpenStack resource types (such as instances, floating ips, volumes, security groups, users, etc), as well as some more advanced functionality such as instance high availability, instance autoscaling, and nested stacks. By providing very tight integration with other OpenStack core projects, all OpenStack core projects could receive a larger user base.
    *   Allow deployers to integrate with Heat directly or by adding custom plugins.

#### 2\. Feature Details

*   A Heat template describes the infrastructure for a cloud application in a text file that is readable and writable by humans, and can be checked into version control, diffed, &c.
*   Infrastructure resources that can be described include: servers, floating ips, volumes, security groups, users, etc.
*   Heat also provides an autoscaling service that integrates with Ceilometer, so you can include a scaling group as a resource in a template.
*   Templates can also specify the relationships between resources (e.g. this volume is connected to this server). This enables Heat to call out to the OpenStack APIs to create all of your infrastructure in the correct order to completely launch your application.
*   Heat manages the whole lifecycle of the application - when you need to change your infrastructure, simply modify the template and use it to update your existing stack. Heat knows how to make the necessary changes. It will delete all of the resources when you are finished with the application, too.
*   Heat primarily manages infrastructure, but the templates integrate well with software configuration management tools such as Puppet and Chef. The Heat team is working on providing even better integration between infrastructure and software.

#### 3\. Service Components

*   heat The heat tool is a CLI which communicates with heat-api and heat-api-cfn
*   heat-api The heat-api component provides an OpenStack-native REST API that processes API requests by sending them to the heat-engine over RPC.
*   heat-api-cfn The heat-api-cfn component provides an AWS Query API that is compatible with AWS CloudFormation and processes API requests by sending them to the heat-engine over RPC.
*   heat-engine The heat engine’s main responsibility is to orchestrate the launching of templates and provide events back to the API consumer.

#### 4\. Evolving

[Blueprints][4]

#### 5\. Installation

[Openstack Heat: installation][5]

#### 6\. HowTo

[OpenStack Heat: Glossary][6]  
[OpenStack Heat: Template Specification][7]  
[OpenStack Heat: Give It A Try][8]  
[OpenStack Heat: AutoScaling][9]

#### 7\. Plugin Development

[reference][10]

#### 8\. Architecture

*To be continued*

 [1]: http://aws.amazon.com/documentation/cloudformation/
 [2]: https://wiki.openstack.org/wiki/Heat
 [3]: http://git.openstack.org/cgit/openstack/governance/tree/reference/programs.yaml
 [4]: https://blueprints.launchpad.net/heat/
 [5]: /openstack/2014/10/15/openstack-heat-installation.html
 [6]: /openstack/2014/11/12/openstack-heat-glossary.html
 [7]: /openstack/2014/10/18/openstack-heat-template-specification.html
 [8]: /openstack/2014/11/12/openstack-heat-give-it-a-try.html
 [9]: /openstack/2014/11/14/openstack-heat-autoscaling.html
 [10]: http://docs.openstack.org/developer/heat/pluginguide.html
