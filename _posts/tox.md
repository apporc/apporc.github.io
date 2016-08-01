### 概览

tox 是 python 单元测试方面的工具，使用它可以方便地设定一个预期的 virtualenv
列表，并自动在每个 virtualenv 下进行指定的安装与测试工作。

tox 的使用步骤概括一下就是：

*   安装 tox
*   生成 tox 配置文件，在其中指定 virtualenv 列表以及每个 virtualenv 环境
下要运行的测试命令
*   由 tox 命令自动创建准备 virtualenv，并在其中运行指定的测试命令

### 快速开始

下面来一起试用一下 tox。

1.  安装 tox

    可以使用 `pip/easy_install` 来安装 tox，或者使用各 Linux 发行版的包管理器：

        pip install tox
        easy_install tox

2.  创建配置文件

    在代码根目录下执行命令 tox-quickstart 并回答相应问题，可以自动生成一个
    tox.ini 文件。

        $ tox-quickstart 

        What Python versions do you want to test against? Choices:
            [1] py27
            [2] py27, py33
            [3] (All versions) py26, py27, py32, py33, py34, py35, pypy, jython
            [4] Choose each one-by-one
        > Enter the number of your choice [3]: 2

        What command should be used to test your project -- examples:
            - py.test
            - python setup.py test
            - nosetests package.module
            - trial package.module
        > Command to run to test project [{envpython} setup.py test]: 

        What extra dependencies do your tests have?
        > Comma-separated list of dependencies [ ]: 

        Finished: A tox.ini file has been created.

    生成的 tox.ini 如下：


        [tox]
        envlist = py27, py33

        [testenv]
        commands = {envpython} setup.py test
        deps =

    在以上示例中，我选择的 virtualenv 列表为 py27 和 py33，这两个 virtualenv 分
    别对应 python 2.7 和 3.3 两个版本。virtualenv 的列表在 tox.ini 是通过 [tox]
    下的 envlist 选项来指定的；  
    在这两个 virtualenv 中预计要运行的测试命令为 `python setup.py test`，通过
    [testenv] 下的 commands 选项来指定；  
    另外 [testenv] 下还有一个选项 deps 留空了，这里实际可以指定 commands 选项
    中的命令所需的依赖包，创建 virtualenv 时 tox 会通过 pip 自动安装依赖包；
    tox 配置文件中允许使用一些环境变量，例如 {envpython}，代指 virtualenv 中的
    python 解释器路径，这些变量由 tox 来解释。

3. 发起测试

    在代码根目录执行 `tox` 就可以发起测试了。  
    测试的过程中， tox 会首先在 .tox 目录下分别建立两个 virtualenv，即 py27 和
    py33，并在其中安装 deps 选项指定的依赖包，最后使用 commands 选项指定的命令
    发起单元测试。  
    通过 `tox -e<env name>` 也可以发起单一 virtualenv 下的测试。

### 常用特性

1.  对不同 virtualenv 的命令进行定制

    在 quickstart 的示例配置之中，[testenv] 下的配置是所有 virtualenv 共享的，
    对于不同 virtualenv 想要指定不同的一些配置，就需要特别指定：

        [tox]
        envlist = py27, py33

        [testenv]
        commands = {envpython} setup.py test
        deps =

        [testenv:py33]
        commands = nosetests package.module

    以上创建了一个新的配置段 [testenv:py33]，其中指定对于 py33 使用
    `nosetests package.module` 来发起单元测试，对于 py27 则仍然使用 [testenv]
    中的默认值。在 py33 所独有的配置段 [testenv:py33] 之中，可以选择覆盖掉
    [testenv] 中的任意一个配置项，对于未覆盖的配置项，则使用 [testenv] 中的
    默认值，例如 deps，由于在 [testenv:py33] 中未指定，所以沿袭 [testenv] 中
    的值。

2.  使用依赖文件来指定依赖 

    以上已经提到通过 deps 选项可以指定发起单元测试所需依赖

        deps = testtools
               pbr

    但在实际的项目中，依赖的列表往往是很长的，tox 支持将一个文本文件作为依赖
    包的列表来指定。

        deps =
            -r{toxinidir}/requirements.txt
            -r{toxinidir}/test-requirements.txt

    {toxinidir} 也是一个环境变量，代表的意义为 "tox.ini 文件所在的目录"。

3.  virtualenv 中调用外部命令

    默认情况下，在 virtualenv 中不能使用外部安装的命令，这本来是为了命令环境的
    隔离，但有些情况下，我们会想要使用外部命令。例如，在对代码对格式检查时，
    我们不会想要在每个 virtualenv 中都安装一遍 flake8，只需调用外部环境中唯一
    的一份 flake8 即可。

        whitelist_externals =
            flake8

4.  skipsdist and usedevelop

    默认情况下，tox 会对当前项目执行 `python setup.py install` 以安装到 
    virtualenv 中，这包含了 `python setup.py sdist` 打包操作，对于大一些的
    项目，这会是一个费时的过程。当我们在调试代码并频繁执行单元测试的时候，我们
    不会想要每次都跑一遍打包操作。此时的办法是使用 `python setup.py develop` 方
    式来进行源码安装。tox 为此提供了便利的参数 skipsdist 和 usedevelop。

        [tox]
        skipsdist = True
        envlist = py27, py33

        [testenv]
        usedevelop = True
        commands = {envpython} setup.py test
        deps =

        [testenv:py33]
        commands = nosetests package.module


5.  代码检查

    执行代码格式检查通常是单元测试的一个组成部分，使用 tox 也可以方便的实现。
    在 tox 限定的这些 (py26, py27, py32, py33, py34, py35, pypy, jython) 
    virtualenv 之外，我们可以任意定义其它 virtualenv，就比如 pep8：

        [tox]
        skipsdist = True
        envlist = pep8

        [testenv:pep8]
        usedevelop = False
        deps =
        whitelist_externals =
            flake8
        commands =
            flake8

    这就指定创建 pep8 这个 virtualenv，并在其中使用 flake8 命令来执行代码检查。
    注意，代码检查并不需要安装软件包，也不需要安装任何依赖，flake8 使用外部的
    命令。

6.  与 jenkins

    tox 支持使用一个额外的 [tox:jenkins] 参数来替换 [tox] 中指定的内容。当检测
    到是在 jenkins 中调用 tox 时，tox 自动使用 [tox:jenkins] 中的配置，而不是
    [tox]。

7.  指定 pip 源

    tox 使用 pip 来安装依赖包时，默认使用的源为 http://pypi.python.org/simple
    然而，也允许自定义源地址：

        [tox]
        indexserver =
            default = http://mypypi.org

### 参考资料

除了以上提到的参数，tox 还有众多的[其它配置项][1]，在使用时可自行查阅。  
tox 官网也提供了广泛的[使用示例][2]。

 [1]: https://testrun.org/tox/latest/config.html#
 [2]: https://testrun.org/tox/latest/examples.html
