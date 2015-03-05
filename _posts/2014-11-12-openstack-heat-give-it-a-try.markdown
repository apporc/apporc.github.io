---
layout: post
title:  "Openstack Heat : Give it a try"
date:   2014-11-12 12:05:00
categories: OpenStack
---
1.  环境要求  
    一个正常工作的openstack环境，　带有heat支持;  
    [metadata-server][1], 即http://169.254.169.254是正常工作的;  
    打算使用的镜像模板中安装了[heat-cfntools][2]

2.  hello_world 测试  
    测试模板: [Hello World Template][3]  
    参数:
    
        parameters:
        key_name:
            type: string
            description: Name of an existing key pair to use for the server
            constraints:
            - custom_constraint: nova.keypair
        flavor:
            type: string
            description: Flavor for the server to be created
            default: m1.small
            constraints:
            - custom_constraint: nova.flavor
        image:
            type: string
            description: Image ID or image name to use for the server
            constraints:
            - custom_constraint: glance.image
        admin_pass:
            type: string
            description: Admin password
            hidden: true
            constraints:
            - length: { min: 6, max: 8 }
                description: Password length must be between 6 and 8 characters
            - allowed_pattern: "[a-zA-Z0-9]+"
                description: Password must consist of characters and numbers only
            - allowed_pattern: "[A-Z]+[a-zA-Z0-9]*"
                description: Password must start with an uppercase character
        db_port:
            type: number
            description: Database port number
            default: 50000
            constraints:
            - range: { min: 40000, max: 60000 }
                description: Port number must be between 40000 and 60000
        
    
    这里是建立一个虚拟机实例所必须的几个参数, key_name是ssh公钥, flavor是虚拟机实例的规格, image为glance中的镜像. 参看这三个参数的constraints, 我们知道三个参数的合法值必须为openstack环境中已有的资源. admin_pass为虚拟机实例root的密码，这里提出了几个限定，　包括长度必须在6-8之间，只包含数字和字母，　且必须以大写字母开头. 没有指定default的参数为必填, 对于有constraints限定的参数, 需要满足限定.  
    注：使admin_pass生效，　需要在compute节点nova.conf中开启inject_password=true[参考链接][4]
    
    内容：
    
        resources:
        server:
            type: OS::Nova::Server
            properties:
            key_name: { get_param: key_name }
            image: { get_param: image }
            flavor: { get_param: flavor }
            admin_pass: { get_param: admin_pass }
            user_data:
                str_replace:
                template: |
                    #!/bin/bash
                    echo db_port
                params:
                    db_port: { get_param: db_port }
        
    
    这里用到了OS::Nova::Server这一资源类型, 详细情况请参考<http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Nova::Server> 这里通过使用get_param这一个Intrinsic Function来获取参数值, 填充进OS::Nova::Server这一类型的property中. 值得注意的是user_data这一个property, 此中的信息是用来传递给虚拟机中的cloud-init去初始化一个实例的配置的. 但该hello_world只是演示了打印一个db_port变量. str_replace意味着template中的db_port将被params中的变量替换.
    
    实测发现，　在不指定networks的情况下，heat默认为虚拟机实例分配的网络是基于ext-net的, 而demo用户并没有使用ext-net的权限。　所以我们需要在此手动指定networks:
    
        networks: [{"network": "demo-net"}]
        
    
    输出:
    
        outputs:
        server_networks:
            description: The networks of the deployed server
            value: { get_attr: [server, networks] }
        
    
    这里使用get_attr这一Intrinsic Function获取已建实例的网络地址信息，反馈给客户.
    
    命令:
    
        heat stack-create helloworld --template-file=hello_world.yaml
        --parameters="image=fedora-heat-cfntools;key_name=test;flavor=m1.small;admin_pass=ABC123;db_port=50000"
        heat stack-list
        heat event-list helloworld
        heat stack-show helloworld
        

3.  进阶测试  
    测试模板: [2 nodes wordpress][5]
    
    参数:
    
        key_name:
            type: string
            description : Name of a KeyPair to enable SSH access to the instance
            default: test_key
        
        instance_type:
            type: string
            description: Instance type for web and DB servers
            default: m1.small
            constraints:
            - allowed_values: [m1.tiny, m1.small, m1.medium, m1.large, m1.xlarge]
                description: instance_type must be a valid instance type
        
        image_id:
            type: string
            description: >
            Name or ID of the image to use for the WordPress server.
            Recommended values are fedora-20.i386 or fedora-20.x86_64;
            get them from http://cloud.fedoraproject.org/fedora-20.i386.qcow2
            or http://cloud.fedoraproject.org/fedora-20.x86_64.qcow2 .
            default: fedora-20.x86_64
        
        db_name:
            type: string
            description: WordPress database name
            default: wordpress
            constraints:
            - length: { min: 1, max: 64 }
                description: db_name must be between 1 and 64 characters
            - allowed_pattern: '[a-zA-Z][a-zA-Z0-9]*'
                description: >
                db_name must begin with a letter and contain only alphanumeric
                characters
        
        db_username:
            type: string
            description: The WordPress database admin account username
            default: admin
            hidden: true
            constraints:
            - length: { min: 1, max: 16 }
                description: db_username must be between 1 and 16 characters
            - allowed_pattern: '[a-zA-Z][a-zA-Z0-9]*'
                description: >
                db_username must begin with a letter and contain only alphanumeric
                characters
        
        db_password:
            type: string
            description: The WordPress database admin account password
            default: admin
            hidden: true
            constraints:
            - length: { min: 1, max: 41 }
                description: db_password must be between 1 and 41 characters
            - allowed_pattern: '[a-zA-Z0-9]*'
                description: db_password must contain only alphanumeric characters
        
        db_root_password:
            type: string
            description: Root password for MySQL
            default: admin
            hidden: true
            constraints:
            - length: { min: 1, max: 41 }
                description: db_root_password must be between 1 and 41 characters
            - allowed_pattern: '[a-zA-Z0-9]*'
                description: db_root_password must contain only alphanumeric characters
        
    
    除了key_name, image_id, instance_type这几个创建虚拟机必要的参数之外，　还有db_name, db_username, db_password, db_root_password这几个参数，　用于在虚拟机中创建wordpress 数据库
    
    内容:
    
        resources:
        DatabaseServer:
            type: OS::Nova::Server
            properties:
            image: { get_param: image_id }
            flavor: { get_param: instance_type }
            key_name: { get_param: key_name }
            user_data:
                str_replace:
                template: |
                    #!/bin/bash -v
        
                    yum -y install mariadb mariadb-server
                    touch /var/log/mariadb/mariadb.log
                    chown mysql.mysql /var/log/mariadb/mariadb.log
                    systemctl start mariadb.service
        
                    # Setup MySQL root password and create a user
                    mysqladmin -u root password db_rootpassword
                    cat << EOF | mysql -u root --password=db_rootpassword
                    CREATE DATABASE db_name;
                    GRANT ALL PRIVILEGES ON db_name.* TO "db_user"@"%"
                    IDENTIFIED BY "db_password";
                    FLUSH PRIVILEGES;
                    EXIT
                    EOF
                params:
                    db_rootpassword: { get_param: db_root_password }
                    db_name: { get_param: db_name }
                    db_user: { get_param: db_username }
                    db_password: { get_param: db_password }
        
        WebServer:
            type: OS::Nova::Server
            properties:
            image: { get_param: image_id }
            flavor: { get_param: instance_type }
            key_name: { get_param: key_name }
            user_data:
                str_replace:
                template: |
                    #!/bin/bash -v
        
                    yum -y install httpd wordpress
        
                    sed -i "/Deny from All/d" /etc/httpd/conf.d/wordpress.conf
                    sed -i "s/Require local/Require all granted/" /etc/httpd/conf.d/wordpress.conf
                    sed -i s/database_name_here/db_name/ /etc/wordpress/wp-config.php
                    sed -i s/username_here/db_user/      /etc/wordpress/wp-config.php
                    sed -i s/password_here/db_password/  /etc/wordpress/wp-config.php
                    sed -i s/localhost/db_ipaddr/        /etc/wordpress/wp-config.php
                    setenforce 0 # Otherwise net traffic with DB is disabled
                    systemctl start httpd.service
                params:
                    db_rootpassword: { get_param: db_root_password }
                    db_name: { get_param: db_name }
                    db_user: { get_param: db_username }
                    db_password: { get_param: db_password }
                    db_ipaddr: { get_attr: [DatabaseServer, networks, demo-net, 0] }
        
    
    此模板中将wordpress的数据库与web分在两个虚拟机实例中. user_data中则提供的丰富的信息用于配置数据库和web服务 与hello_world测试时相同，　我们需要为虚拟机手动指定networks,　并且由于demo-net的内容我们不能直接访问，　所以这里 添加一个floatingip的资源:
    
        networks: [{"network": "demo-net"}]
        
        Floatingip:
            type: OS::Nova::FloatingIP
            properties:
            pool: ext-net
        
        Floatingipassociation:
            type: OS::Nova::FloatingIPAssociation
            properties:
            floating_ip: {get_resource: Floatingip}
            server_id: {get_resource: WebServer}
        
    
    注: check nova floating-ip-pool-list to see the pool name of floating ip
    
    输出:
    
        outputs:
        WebsiteURL:
            description: URL for Wordpress wiki
            value:
        
            str_replace:
                template: |
                    web: http://$wordpress/wordpress
                    server: $host
                params:
                    $wordpress: { get_attr: [Floatingip, ip] }
                    $host: { get_attr: [WebServer, networks, demo-net, 0] }
        
    
    反馈这一wordpress部署的web访问地址, 注意get_attr的使用方式, 由于我们使用了floatingip, 所以这里也有所更改
    
    命令:
    
        heat stack-create wordpress --template-file=WordPress_2_Instances.yaml
        --parameters="key_name=test;image_id=fedora-heat-cfntools;instance_type=m1.small;db_name=wordpress;db_username=wordpress;db_password=abc123;db_root_password=abc123"
        heat stack-list
        heat event-list wordpress
        heat stack-show wordpress
        
    
    注: 参数越来越多? 使用[Environments][6]

 [1]: http://docs.openstack.org/icehouse/install-guide/install/yum/content/neutron-ml2-network-node.html
 [2]: /openstack/2014/10/15/openstack-heat-installation.html
 [3]: https://github.com/openstack/heat-templates/blob/master/hot/hello_world.yaml
 [4]: https://kimizhang.wordpress.com/2014/03/18/how-to-inject-filemetassh-keyroot-passworduserdataconfig-drive-to-a-vm-during-nova-boot/
 [5]: https://github.com/openstack/heat-templates/blob/master/hot/F20/WordPress_2_Instances.yaml
 [6]: http://docs.openstack.org/developer/heat/template_guide/environment.html#environments
