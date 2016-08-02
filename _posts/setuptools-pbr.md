### 概览

setuptools 是针对 python 项目的一个打包工具，使用它可以方便地对 python 项目进行
打包安装。要使用 setuptools 只需要在安装脚本中调用其提供的 setup 方法，项目的信
息都通过高度可定制的参数传递给该方法。当调用安装脚本的时候，可以通过传递诸如
build, install 等参数直接进行构建与安装的操作。setuptools 是对 python 标准库中的
[distutils][4] 的加强。

pbr 是一个对 setuptools 加强的插件，通过它可以将 setup 函数所需的参数放在一个统
一的配置文件中集中管理，此外它也带有自动化管理项目版本的功能。pbr 来自于
[openstack][3] 社区。

### 快速开始

1.  安装 setuptools 和 pbr

        pip install setuptools pbr

2.  创建安装脚本 setup.py

    首先需要为项目建立一个安装脚本，脚本的名称推荐 setup.py，因为许多 python 持
    续集成工具都默认使用这个名称，虽然理论上其可以随意设定。

        #!/bin/env python
        # setup.py
        from setuptools import setup, find_packages
        setup(
            name = "HelloWorld",
            version = "0.1",
            packages = find_packages('hello_world'),
        )

    如此一个安装脚本就完成了，注意其中的 `find_packages()` 方法，它可能比较 magic
    ，它负责自动地从项目代码中找到需要打包安装的软件包。也就是说那些预计打包的
    代码需要是整理在一个 python 包(有 `__init__.py` 文件的目录)之中的，示例中的
    包即 hello\_world。

    使用 setuptools 的项目普通采用的文件组织方式是像这样的：

        .
        ├── hello_world
        │   ├── __init__.py
        │   └── tests
        │       └── __init__.py
        ├── README.md
        ├── setup.cfg
        └── setup.py

    hello\_world 就是一个等待被 setuptools 加入的包，其中有后面会提到的单元测试代
    码目录 tests/ 以及 pbr 会用到的 setup.cfg 文件。

    如果不打算使用 pbr，那么现在已经可以使用这个安装脚本了。例如：

    *   构建：  `python setup.py build`
    *   安装：  `python setup.py install`
    *   创建源码包：    `python setup.py sdist`
    *   创建二进制包：  `python setup.py bdist`
    *   上传到 pypi：   `python setup.py upload`

3.  使用 setup.cfg 配置文件

    对于一个实际项目来说，项目的配置参数可以是一个非常可观的规模，pbr 可以为日益
    臃肿的 setup.py 瘦身。使用 pbr 时，绝大多数配置参数可以从 setup.py 中转移至
    一个 setup.cfg 之中。

    更改 setup.py：

        setup(
            setup_requires=['pbr', 'setuptools'],
            pbr=True,
        )
    通过 setup\_requires 指定 setup.py 的运行依赖包，通过 pbr=True 启动 pbr。

    最后把原来 setup.py 中的配置项转移到 setup.cfg 之中：

        [metadata]
        name = hello_world
        version = 0.0.1

        [global]
        setup-hooks =
            pbr.hooks.setup_hook

        [files]
        packages =
            hello_world

    以上就完成了对 pbr 的引入，需要注意的是 pbr 对于 setup.cfg 中的 version 有一
    套自己的解释，下文再提。

    注意：使用 pbr 的项目必须是由 git 管理的，如果你的项目还没有加入 git 版本管理
    ，那你还等什么？

4.  使用安装脚本

    仅仅是在安装脚本中调用了 setup 函数，我们已经有了一个功能完善的安装脚本。
    查看安装脚本的使用帮助：

        python setup.py --help
        python setup.py --help-commands

    如果在第 2 步的时候你曾测试了 `python setup.py build` 等命令，你可能会发现
    现在同样的命令会产生更多的临时文件，例如 AUTHORS, ChangeLog 等，这些就是
    pbr 的工作成果，pbr 会通过 git 自动生成项目作者名单以及变更记录等，另外它
    连版本号也自动管理了，记得上文提到的 version 参数吧？

### 常用特性

下面说一下实际应用中经常会用到的特性，便于对 setuptools 的能力有一个基本把握。
至于更高级的功能，请移步参考资料章节。

对于以下特性，我们专注于使用 pbr 时其对应在 setup.cfg 中的配置方法，不使用 pbr
时 setup.py 中的配置方法请参考官方文档。两种方法配置项的名称基本上保持了一致，
所以应该很容易查阅到。

1.  可执行文件

    安装一个软件包的时候，通常我们会得到一些可执行的命令（只提供代码库的例外）
    ，setuptools 可以很容易地达成这一点。

    假设我们有 hello\_world/hello\_world.py：

        # hello_world.py
        def main():
            print("Hello World!")

    编辑 setup.cfg 增加 entry\_points 选项：

        [entry_points]
        console_scripts =
            hello_world  =  hello_world.hello_world:main

    使用 `python setup.py install` 安装之后，我们就会在系统中得到一个名称为
    hello\_world 的命令，安装过程中 setuptools 为我们在 /usr/bin 下创建该命令
    文件，执行它：

        $ hello_world
        Hello World!

2.  包含静态文件

    对于 python 代码文件，使用 find\_packages() 可以自动打包进来，但是其它类型
    的文件就不行了，例如图片，html 文件等。对于此类静态文件，需要用到
    MANIFEST.in 文件，例如：

        include scripts/*
        recursive-include static *.html *.css

    以上指示 setuptools 打包 scripts 下面的所有文件，以及 static 下面的 html
    和 css 文件。

    对于这种方式打包的文件，我们是不知道其文件路径的，因为安装过程中，
    setuptools 会根据其安装环境来动态适应。这些文件，需要使用 pkg\_resources
    模块来获取。例如：

        from pkg_resources import resource_stream

        text = resource_stream('hello_world',
                               'static/hello_world.html').read()

    更多 pkg\_resources 模块的使用方法参见 [pkg\_resource][5]。

    除了以上的方式来包含静态文件之外，你可能还需要将一些默认的配置文件放置到
    系统的标准配置文件目录下，例如 Linux 下的 /etc/，此时需要使用 data\_files
    配置项：

        [files]
        packages = 
            hello_world

        data_files =
            # 假设我们在 etc 下有 hello_world.conf 这个配置文件，想要安装到
            # 系统的 /etc/hello_world/ 目录之下
            etc/hello_world = etc/hello_world.conf

    注意：使用以上配置的同时，在执行安装脚本时也要传递 `--prefix=/` 选项，否
    则默认会把配置文件安装到 /usr/etc/hello\_world/ 之下。


3.  集成单元测试

    setuptools 能够与单元测试模块 unittest 以及 testtools 进行集成，上文之中
    在描述标准的代码文件组织结构的时候，预留了 tests 目录。假设 tests 包下有
    一些单元测试模块，我们可以这样与 setuptools 进行集成：
    
        # setup.py
        setup(
            setup_requires=['pbr', 'setuptools'],
            pbr=True,
            test_suite='hello_world.tests',
        )

    通过为 setuptools 指定 test\_suite 参数，我们让 setuptools 知道了我们的单元
    测试代码的位置。只需要执行以下命令即可发起单元测试：

        python setup.py test

    这部分功能是不涉及 pbr 模块的。

4.  上传软件包到 pypi

    通过 setuptools 可以方便地将项目打包上传到 pypi 服务器，方便后续使用 pip 来
    安装。

        python setup.py sdist upload

### 参考资料

[setuptools][1]
[pbr][2]


 [1]: https://setuptools.readthedocs.io/en/latest/setuptools.html
 [2]: http://docs.openstack.org/developer/pbr/
 [3]: http://www.openstack.org/software/
 [4]: https://docs.python.org/2.7/library/distutils.html
 [5]: http://peak.telecommunity.com/DevCenter/PkgResources#resourcemanager-api
