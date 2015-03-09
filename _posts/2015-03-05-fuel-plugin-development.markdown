---
layout: post
title:  "Fuel Plugin Development"
date:   2015-03-05 17:25:00
categories: OpenStack
---

#### Table of Contents

1. [概述](#section)
2. [开发插件](#section-1)
    * [环境准备](#section-2)
    * [插件创建与编译](#section-3)
    * [插件代码结构](#section-4)
4. [使用插件](#section-5)
5. [插件认证与提交](#section-6)

## 概述

从Fuel6.0开始，Fuel支持插件机制，对Fuel功能的补充、打补
丁、特殊硬件支持等，都可以通过制作插件的方式来完成。例
如，默认情况下fuel部署的openstack环境不支持neutron的lba
as，通过添加一个lbaas插件的方式就可以支持起来。

## 开发插件

### 环境准备

* 一些依赖软件包的安装

    * Ubuntu 12.04 下：  

            sudo apt-get install createrepo rpm dpkg-dev
    * CentOS 6.5　下：  

            yum install createrepo rpm dpkg-devel

* 安装Fuel Plug-in Builder(即fpb命令行工具)

    * 通过pip安装  

            easy_install pip
            pip install fuel-plugin-builder

    * 手动源码安装  

            git clone https://github.com/stackforge/fuel-plugins
            cd fuel-plugins/fuel_plugin_builder/
            sudo python setup.py install

    * fpb工具使用  
        以上安装完后，系统中会有一个fpb命令，来协助fuel插件的开发。
        详情查看fpb --help。

### 插件创建与编译

* 创建代码结构

        fpb --create fuel_plugin_name
    该命令会在fuel_plugin_name目录下创建默认的插件代码结构，详情
    参见[代码结构](#插件代码结构)

* 构建插件

        fpb --build fuel_plugin_name
    命令成功之后，会生成一个
    fuel_plugin_name/fuel_plugin_name-0.1.0.fp文件。该文件即是fuel
    插件的形式。

### 插件代码结构

fuel插件的代码结构如下：

    fuel_plugin_name/
    ├── deployment_scripts
    │   └── deploy.sh
    ├── environment_config.yaml
    ├── metadata.yaml
    ├── pre_build_hook
    ├── repositories
    │   ├── centos
    │   └── ubuntu
    └── tasks.yaml

* tasks.yaml  
    说明该插件在何时何处怎样执行，示例：

        - role: ['controller']
        stage: post_deployment
        type: shell
        parameters:
            cmd: ./deploy.sh
            timeout: 42
        - role: ['compute', 'controller']
        stage: post_deployment
        type: puppet
        parameters:
            puppet_manifest: puppet/manifests/site.pp
            puppet_modules: puppet/modules
            timeout: 360

    * role 定义了该插件在哪些角色的机器上应被执行，是一个列表，也可
      以用'*'代表在所有角色的机器上都执行。
    * stage 定义了该插件在什么时候被执行，只有两个可选项，
      pre_deployment或者post_deployment，分别是在部署前和部署后。
    * type 有两种，shell或者puppet。
    * 如果是puppet类型的插件， 把site.pp放到
      deployment_scripts/puppet/manifests/下，把模块放到
      deployment_scripts/puppet/modules下。
    * 不管是哪种type，都可以指定一个超时时间，默认的超时尚不清
      楚是多少。

* environment_config.yaml
    该文件定义了界面上会显示的配置项，这些配置项会最终写到每个
    节点的/etc/astute.yaml中。以供插件部署时使用。
* metadata.yaml
    该文件定义了模块的信息。包括了模块的名称、版本、介绍以及支
    持的发行版等信息。

## 使用插件

* 安装
    复制fuel插件至fuel管理节点上，执行如下命令安装：

        fuel plugins --install fuel_plugin_name.fp
* 配置
    在fuel界面上Settings标签，启用相应插件并设置配置项即可。

## 插件认证与提交

编写的插件可以提交Mirantis进行认证，发送fuel-plugin@mirantis.com即可。

