### 概览

oslo.config 是用于从命令行或配置文件解析配置参数的框架，来自于万能的 OpenStack
社区。作为 oslo 项目的子项目，可以通用在任何 python 程序中。

oslo.config 的主要特性包括：

*   参数的类型限定
*   同时管理命令行与配置文件(ini)
*   自动生成示例配置文件
*   支持参数分组
*   运行时重新载入配置

### 快速开始

1.  安装 oslo.config

        pip install oslo.config

2.  定义参数

        #!/usr/bin/python
        # test_oslo_config.py
        from oslo_config import cfg
        from oslo_config import types

        PortType = types.Integer(1, 65535)

        common_opts = [
            cfg.StrOpt('bind_host',
                    default='0.0.0.0',
                    help='IP address to listen on.'),
            cfg.Opt('bind_port',
                    type=PortType,
                    default=9292,
                    help='Port number to listen on.')
        ]

3.  注册参数

        # oslo.config 默认维护了一个 ConfigOpts 类型的全局变量 CONF。
        # 注册参数，以及后续获取参数，都是可以通过 CONF。
        # 当然，你也可以创建管理自己的 ConfigOpts 对象。
        CONF = cfg.CONF
        CONF.register_opts(common_opts)
        CONF.register_cli_opts(common_opts)

4.  使用参数

        import sys
        if __name__ == '__main__':
            # 需要将命令行参数传递给 CONF。
            CONF(sys.argv[1:])
            print("bind_host: %s, bind_port: %s" % (CONF.bind.bind_host,
                                                    CONF.bind.bind_port))

    运行一下 test\_oslo\_config.py

        $ ./test_oslo_config.py --bind_host localhost --bind_port 8080
        bind_host: localhost, bind_port: 8080

    获取命令帮助

        $ ./test_oslo_config.py -h
        usage: test_oslo_config.py [-h] [--bind_host BIND_HOST]
                                [--bind_port BIND_PORT] [--config-dir DIR]
                                [--config-file PATH]

        optional arguments:
        -h, --help            show this help message and exit
        --bind_host BIND_HOST
                                IP address to listen on.
        --bind_port BIND_PORT
                                Port number to listen on.
        --config-dir DIR      Path to a config directory to pull *.conf files from.
                                This file set is sorted, so as to provide a
                                predictable parse order if individual options are
                                over-ridden. The set is parsed after the file(s)
                                specified via previous --config-file, arguments hence
                                over-ridden options in the directory take precedence.
        --config-file PATH    Path to a config file to use. Multiple config files
                                can be specified, with values in later files taking
                                precedence. Defaults to None.

    `--config-dir` 与 `--config-file` 是 oslo.config 默认保留的参数，分别用于
    指定配置文件目录与名称。

### 常用特性

1.   参数的类型限定

    在快速开始章节，已经看到 olso.config 支持对参数的类型进行限定，并且有两种方
    式来实现：

        # test_oslo_config.py
        PortType = types.Integer(1, 65535)

        cfg.StrOpt('bind_host',
                default='0.0.0.0',
                help='IP address to listen on.'),
        cfg.Opt('bind_port',
                type=PortType,
                default=9292,
                help='Port number to listen on.')

    第一种方式是使用 cfg.StrOpt，cfg.IntOpt 等带有类型的参数类；另一种方式是在
    初始化 cfg.Opt 时传递 type 参数。

    更多参数类型，请参考 [option-types][1] 和 [types][2]。

    使用快速开始章节中的 test\_oslo\_config.py，传递一个错误类型的参数：

        $ ./test_oslo_config.py --bind_host localhost --bind_port abc
        argument --bind_port: Invalid Integer(min=1, max=65535) value: abc

2.   同时管理命令行与配置文件(ini)

    test\_oslo\_config.py 中，我们把相同的参数既注册为命令行参数又注册为配置文件
    参数了：

        # test_oslo_config.py
        CONF.register_opts(common_opts)
        CONF.register_cli_opts(common_opts)

    如果，在配置文件与命令行中同时指定一个参数，那么后注册的会覆盖掉先注册掉的
    参数。本例中，命令行参数先注册，所以命令行中的参数会覆盖掉文件中的参数：

        # test.conf
        [DEFAULT]
        bind_host=test.com
        bind_port=80

        $ ./test_oslo_config.py --config-dir=. --config-file=test.conf
        bind_host: test.com, bind_port: 80
        $ ./test_oslo_config.py --config-dir=. --config-file=test.conf --bind_host=localhost --bind_port=8080
        bind_host: localhost, bind_port: 8080

3.   支持参数分组

    oslo.config 支持对配置参数进行分组，未指定分组的参数在 DEFAULT 分组之中。

        # test_oslo_config.py
        # 创建参数组
        logger_group = cfg.OptGroup(name='logger',
                                title='logger options')

        logger_opts = [
            cfg.BoolOpt('debug',
                        default=False,
                        help='Enable debug log or not.'),
            cfg.BoolOpt('verbose',
                        default=True,
                        help='Enable verbose log or not.')
        ]

        # 注册参数组
        CONF.register_group(logger_group)
        # 注册参数时指定参数组
        CONF.register_opts(logger_opts, logger_group)
        CONF.register_cli_opts(logger_opts, logger_group)

        # 使用参数组中的参数
        print('debug: %s, verbose: %s' % (CONF.logger.debug,
                                          CONF.logger.verbose))

    获取帮助，来查看如何指定参数组中的参数：

        $ ./test_oslo_config.py -h
        usage: test_oslo_config.py [-h] [--bind_host BIND_HOST]
                                [--bind_port BIND_PORT] [--config-dir DIR]
                                [--config-file PATH] [--logger-debug]
                                [--logger-nodebug] [--logger-noverbose]
                                [--logger-verbose]

        optional arguments:
          -h, --help            show this help message and exit
          ... ...
        logger options:
          --logger-debug        Enable debug log or not.
          --logger-nodebug      The inverse of --debug
          --logger-noverbose    The inverse of --verbose
          --logger-verbose      Enable verbose log or not.

    由此可以看出，对于参数组中的参数，需要使用 `--[参数组]-[参数]` 来指定。
    另外，由于 debug 和 verbose 两个参数都是 Bool 类型的，所以对应会有一对
    `--[参数组]-[参数]` 和 `--[参数组]-no[参数]` 存在，分别用于指定 True 和 False。

    在配置文件中指定参数组的话，只需要像下面这样指定即可：

        # test.conf
        [logger]
        debug=false
        verbose=true

4.   自动生成示例配置文件

    每当更新过代码，都需要去手动更新示例配置文件将是令人厌烦的，能够自动化该
    工作是必要的：

    *   首先将参数作整理输出

            # config.py
            from test_oslo_config import common_opts, logger_opts


            def list_opts():
                return [(None, common_opts), ('logger', logger_opts)]

    *   在 setup.cfg 的 entry\_points 栏目下创建 oslo.config.opts (setuptools 
        与 pbr 的使用在此不作描述，参考 [setuptools & pbr][3])

            # setup.cfg
            oslo.config.opts =
                test_oslo_config  =  test_oslo_config.config:list_opts

    *   使用 oslo-config-generator 命令生成示例配置文件

            $ oslo-config-generator --namespace test_oslo_config > test.conf

        test.conf 就是一个示例配置文件。

5.   运行时重新载入配置

    oslo.config 可以在程序运行时重新载入配置参数，使用步骤是
    
    *   将需要运行时重新载入的配置参数设置为 mutable

            cfg.StrOpt('bind_host',
                    default='0.0.0.0',
                    mutable=True,
                    help='IP address to listen on.')

    *   在程序中处理 SIGHUP 信号，当收到信号的时候，调用
        `CONF.mutate_config_files`

    注意：做到以上两条，并不代表新的 bind\_host 参数已经生效了，还需要自己保证
    在程序中去使用新的参数值，而不是仍然在用旧的，具体来说就是要重新获取
    CONF.bind_host 这个变量的值并应用起来。

### 参考资料

[oslo.config][4]

 [1]: http://docs.openstack.org/developer/oslo.config/cfg.html#option-types
 [2]: http://docs.openstack.org/developer/oslo.config/types.html
 [3]: https://blog.apporc.org/2016/08/python-%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%EF%BC%9A-setuptools-pbr/
 [4]: http://docs.openstack.org/developer/oslo.config/
