---
layout: post
title:  "Openstack Heat : installation"
date:   2014-10-15 15:45:00
categories: OpenStack
---
#### 1\. Installation

*   heat installation heat的安装方式比较多， 下面三种方式供参考
    
    *   Manual:
        
        heat的安装脚本已经加到[teststack][1]. 如果是在已有部署上新加heat, 只需执行:
        
            fab install_heat
            
        
        参考:<http://docs.openstack.org/icehouse/install-guide/install/yum/content/heat-install.html>
    
    *   rdo:
        
        answer file中更改下面的配置:
        
            CONFIG_HEAT_INSTALL=y
            CONFIG_HEAT_CFN_INSTALL = y
            
        
        参考:<https://openstack.redhat.com/Docs>
    
    *   devstack:
        
        参考:<http://docs.openstack.org/developer/heat/getting_started/on_devstack.html>

*   ceilometer installation
    
    Heat的许多功能都需要与ceilometer进行配合才能够完成，所以如果要使用heat的全部功能，例如autoscaling, 也需要同时安装ceilometer.
    
    *   Manual: ceilometer的安装脚本已经加到[teststack][1]. 在已有部署上新加ceilometer, 只需执行:
        
            fab install_ceilometer
            
    
    *   rdo:
        
        answer file中更改下面的配置:
        
            CONFIG_CEILOMETER_INSTALL=y
            
        
        参考:<https://openstack.redhat.com/CeilometerQuickStart>
    
    *   devstack:
        
        参考:<http://docs.openstack.org/developer/ceilometer/install/development.html>

#### 2\. Build JEOS image

为了能使用heat的全部功能，需要在在相应实例镜像中安装heat-cfntools. 这其中主要是对cloud-init和[CloudFormation instance helper scripts][2]的支持.

openstack提供了一个工具[diskimage-builder][3]来完成这个工作. 步骤如下:

*   安装diskimage-builder, 并从tripleo-image-elements中拿到heat-cfntools的elements提供给diskimage-builder
    
        pip install git+https://github.com/openstack/diskimage-builder.git
        git clone https://github.com/openstack/tripleo-image-elements.git
        export ELEMENTS_PATH=tripleo-image-elements/elements
        

*   制作镜像 示例创建一个预装了heat-cfntools的fedora x86_64镜像
    
        disk-image-create vm fedora heat-cfntools -a amd64 -o fedora-heat-cfntools
        
    
    将镜像上传到glance即以供heat使用
    
        source ~/.openstack/keystonerc
        glance image-create --name fedora-heat-cfntools --is-public true --disk-format qcow2 --container-format bare < fedora-heat-cfntools.qcow2
        

也可以通过oz来制作该镜像，详细请参考:<http://docs.openstack.org/developer/heat/getting_started/jeos_building.html>

#### 3\. Scaling a heat Deployment

    参考: [http://docs.openstack.org/developer/heat/scale_deployment.html](http://docs.openstack.org/developer/heat/scale_deployment.html)

 [1]: https://github.com/eayunstack/teststack
 [2]: https://wiki.openstack.org/wiki/Heat/HA
 [3]: https://github.com/openstack/diskimage-builder

