---
layout: post
title:  "OpenStack Nova 单元测试模块列表"
date:   2015-05-05 11:56:00
categories: OpenStack
---

#### Table of Contents

1. [概述](#section)
2. [模块列表](#section-1)
    * [testtools](#testtools)
    * [fixtures](#fixtures)
    * [mock](#mock)
    * [mox](#mox)
    * [tox](#tox)


## 概述

OpenStack Nova 单元测试中借用了大量的 python 测试工具模块，目前看到的有：
testtools，fixture，mox，mock，tox，testrepository等。

## 模块列表

### testtools

unittest 是 python 标准库中自带的单元测试框架，python 2.7 及以后版本中为它加入一
些新特性，将这些新特性整理形成了在 python 2.6-3.4 都能工作的 unittest2 模块。
而 testtools 即是对 unittest2 的扩展。

在安装有 testtools 的 python 环境中找到文件
lib/python2.7/site-packages/testtools/testcase.py，可以看到 testtools.TestCase
继承自 unittest2.TestCase。

该模块是由 Twisted 测试框架 Trial 作者所写，根据[pythonTestingToolsTaxonomy][5]
所述，其将 Trial 与 Bazaar 测试框架的一些有用特性移植过来。
至于使用 testtools 的意义，主要包括更灵活的断言方法，调试更方便等，详情请
参考[testtools][1]。

#### 代码示例

testtools 与 unittest 的使用方式基本是一致的，本文只以 testtools 为例，
下面来看示例代码：

    # -- someserver_test.py --
    from testtools import TestCase
    from testtools.content import attach_file, Content
    from testtools.content_type import UTF8_TEXT

    from myproject import SomeServer

    class SomeTestCase(TestCase):

        def setUp(self):
            super(SomeTestCase, self).setUp()
            self.server = SomeServer()
            self.server.start_up()
            self.addCleanup(self.server.shut_down)
            self.addCleanup(attach_file, self.server.logfile, self)

        def attach_log_file(self):
            self.addDetail(
                'log-file',
                Content(UTF8_TEXT,
                        lambda: open(self.server.logfile, 'r').readlines()))

        def test_a_thing(self):
            self.assertEqual("cool", self.server.temperature)

执行测试：

    python -m testtools.run someserver_test.SomeTestCase

要点：
1. 新建用例继承 testtools.TestCase，如示例中 SomeTestCase。
2. 为 SomeTestCase 定义 setUp 方法，为测试做环境准备。
    示例中生成了待测代码 SomeServer 的一个实例 self.server
    对应的，可以使用 tearDown 方法来定义如何清理测试环境。
3. 通过使用 self.addCleanup 定义测试结束或故障退出时如何清理测试环境。
    所添加的函数，会以与添加时的顺序相逆的顺序来执行，该方式比 tearDown 更健壮。
4. self.addDetail 是类似于打日志一样，可以输出一些调试信息，输出信息使用
    Content，是基于 MIME 的内容类型。
5. 将每一项测试写成名称以 test_ 开头的方法，执行测试时会执行符合该条件的方法。

#### 断言与调试工具

每一个测试方法，基本上就是包含一系列断言，来判断代码运行的方式。
unittest 所提供的断言方式，testtools 全部都保留了，只是某一些方法会有调整。
类似于 assertTrue()，assertFalse() 等各类断言方法在此就不一一提及了，详细的列表
可以前往 [unittest][2] 和 [testtools][1] 查询。

在此只记录几个特别的用法：
1. ExpectedException
    结合 with 语句，可以用于判断一断代码会抛出某一种异常

    with testtools.ExpectedException(ZeroDivisionError):
        1/0

2. expectFailure

    testtools.TestCase.expectFailure 来说明预期某一个断言会失败。

3. assertThat

    便于自定义断言

        def test_abs(self):
            result = abs(-7)
            self.assertEqual(result, 7)
            self.assertNotEqual(result, 7)
            self.assertThat(result, testtools.matchers.Equals(7))
            self.assertThat(result, testtools.matchers.Not(Equals(8)))

    Equals 或 Not 等 Matchers 可以自定义。 通常就是新建一个类，并为类定义一个 match
    方法，详情参阅 [testtools][3]

4. Fixtures

    通常当我们需要在不再的测试用例之间进行协调的时候，需要类似下面这样的代码段：

        class SomeTest(TestCase):
            def setUp(self):
                super(SomeTest, self).setUp()
                self.server = Server()
                self.server.setUp()
                self.addCleanup(self.server.tearDown)

    我们会需要调用其它测试用例类的 setUp 函数，当预定何时执行其 tearDown。
    testtools 为此提供了一个更方便的方式 usefixtures：

        class SomeTest(TestCase):
            def setUp(self):
                super(SomeTest, self).setUp()
                self.server = self.useFixture(Server())

    这里 Server 必须为符合 [fixture.Fixtures](fixtures) 协议的对象。

5. skipTest

    可以在检测不符合所需环境的情况下，忽略某些测试

        def test_make_symlink(self):
            symlink = getattr(os, 'symlink', None)
            if symlink is None:
                self.skipTest("No symlink support")
            symlink(whatever, something_else)

6. patch

    可以代码的特定执行期间将某一对象进行替换

        def test_foo(self):
            my_stream = StringIO()
            self.patch(sys, 'stderr', my_stream)
            run_some_code_that_prints_to_stderr()
            self.assertEqual('', my_stream.getvalue())

    使用自定义的 StringIO 类型 my_stream 来替换 sys.stderr，以便于在一定时间内收集
    打印到 stderr 中的信息。

7. getUniqueString 和 getUniqueInteger

    以往测试时，在需要使用一些“魔数”和“字符串”时，都是手动指定，现在通过使用
    getUniqueString 和 getUniqueInteger 来自动生成。

    关于此类方法的详细讨论：[creation methods][4]

#### 如何运行

除了指定模块或测试用例

    python -m testtools.run <module.testcase>

的方式运行测试之外，也可以指定测试代码的路径，然后自动发现测试用例：

    python -m testtools.run discover <filepath>

另外，还有更进阶的一些专门框架来运行测试，随着项目的增大，可以适时选用：

    [testrepository][12]
    Trial
    nose
    unittest2
    zope.testrunner (zope.testing)

### fixtures

前述，测试用例一般都包含一个 setUp 方法来准备测试环境，主要是生成一些测试过程中
会用到的变量与对象。有时候我们会需要在不再的测试用例间共享这些变量与对象，避免
在每一个测试用例中重复定义。所以就有了 fixtures，它制定了一套规范，将测试代码与
环境准备代码分离开来，使得测试代码亦可方便的组织和扩展。

#### 代码示例

定义 fixtures

    import fixtures
    class NoddyFixture(fixtures.Fixture):
        def _setUp(self):
            self.frobnozzle = 42
            self.addCleanup(delattr, self, 'frobnozzle')

fixtures 文档中推荐在 _setUp 方法中定义准备代码，其内建的 setUp 方法会调用
_setUp，不过从 OpenStack 的代码所看到的基本都直接在 setUp 中定义准备代码的。

重用 fixtures

    class ReusingFixture(fixtures.Fixture):
        def _setUp(self):
            self.noddy = self.useFixture(NoddyFixture())

从 unittest.TestCase 或 testtools.TestCase 中使用 fixtures 的时候，需要调用
fixture 对象的 cleanUp 和 setUp 方法。 对于 testtools，其提供了 useFixture
来简化该过程。


    class SomeUnitTest(unittest.TestCase):
        def setUp(self):
            super(SomeTest, self).setUp()
            self.fixture = NoddyFixture()
            self.fixture.setUp()
            self.addCleanup(self.fixture.cleanUp)

    class SomeTesttoolsTest(testtools.TestCase):
        def setUp(self):
            super(SomeTest, self).setUp()
            self.useFixture(NoddyFixture())

fixtures 的详细信息，可能参照 [fixtures][6]

### mock

mock 模块提供的功能是，将 python 对象替换为 Mock 对象，测试中使用到这些对象时，
就会使用相应的 Mock 对象。而这些 Mock 对象会记录下它被怎样的调用了，以备查看。
前述 testtools 提供一个 patch 方法来替换对象， mock 模块也提供了类似方法，另外
mock 模块还负责提供 Mock 对象。

python 2.7 及之前版本，需要单独 import mock 来使用，python 3 之后版本的 unittest
中集成了 mock，在 unittest.mock 中。

mock 模块[详细文档][7]

#### 代码示例

使用 Mock 来替换方法

    class ProductionClass(object):
        def method(self):
            self.something(1, 2, 3)
        def something(self, a, b, c):
            pass
    real = ProductionClass()
    real.something = MagicMock()
    real.method()
    real.something.assert_called_with(1, 2, 3)
    real.something.assert_called_once_with(1, 2, 3)


使用 Mock 替换普通对象

    class ProductionClass(object):
        def closer(self, something):
            something.close()
    real = ProductionClass()
    mock = Mock()
    real.closer(mock)
    mock.close.assert_called_with()
    # mock 的 close 方法不需要我们建立，当 mock 被调用到 close 时其会自动建立

使用 Mock 替换类

    def some_function():
        instance = module.Foo()
        return instance.method()

    with patch('module.Foo') as mock:
        some_function()
        mock.method.assert_called_with()

使用 Mock 的被调用记录

    >>> mock = MagicMock()
    >>> mock.method()
    <MagicMock name='mock.method()' id='...'>
    >>> mock.attribute.method(10, x=53)
    <MagicMock name='mock.attribute.method()' id='...'>
    >>> expected = [call.method(), call.attribute.method(10, x=53)]
    >>> mock.mock_calls
    [call.method(), call.attribute.method(10, x=53)]
    >>> mock.mock_calls == expected
    True
    # 使用 mock_calls 来查看所记录的被调用方法列表与我们预期的一致。

设置 Mock 的返回值以及属性值

    >>> mock = Mock()
    >>> mock.method.return_value = 3
    >>> mock.method()
    3
    >>> mock = Mock(return_value=3)
    >>> mock()
    3
    >>> mock = Mock()
    >>> cursor = mock.connection.cursor.return_value
    >>> cursor.execute.return_value = ['foo']
    >>> mock.connection.cursor().execute("SELECT 1")
    ['foo']

使用 Mock 的 side_effect

    # 当调用 Mock 或其方法时，我们需要每次调用的返回值不同时，使用 side_effect
    >>> vals = {(1, 2): 1, (2, 3): 2}
    >>> def side_effect(*args):
    ...     return vals[args]
    ...
    >>> mock = MagicMock(side_effect=side_effect)
    >>> mock(1, 2)
    1
    >>> mock(2, 3)
    2

根据已有类创建 Mock，只有类中已有的方法会在 Mock 中出现

    # 传递 spec 给 Mock，SomeClass 所没有的方法在 Mock 中将不会存在。
    mock = Mock(spec=SomeClass)
    mock.some_unknown_method()
    # 调用不存在的方法会抛出 AttributeError

#### 使用 mock 模块的 patch

使用 patch.object 来 patch 方法，patch 来 patch 模块，既可以使用修饰符，也可以
使用 with 语句。

    >>> class MyTest(unittest2.TestCase):
    ...     # 使用自定义对象来替换原有对象
    ...     @patch.object(SomeClass, 'attribute', sentinel.attribute)
    ...     # 使用自动的 Mock 对象来替换原有对象，Mock 对象会被作为参数自动传给
    ...     # 测试函数，注意顺序与修饰时顺序相反。
    ...     @patch('package.module.ClassName1')
    ...     @patch('package.module.ClassName2')    
    ...     def test_something(self, MockClass2, MockClass1):
    ...         self.assertEqual(SomeClass.attribute, sentinel.attribute)
    ...         self.assertTrue(package.module.ClassName1 is MockClass1)
    ...         self.assertTrue(package.module.ClassName2 is MockClass2)
    ...
    >>> original = SomeClass.attribute
    >>> MyTest('test_something').test_something()
    >>> assert SomeClass.attribute == original
    >>> class ProductionClass(object):
    ...     def method(self):
    ...         pass
    ...
    >>> with patch.object(ProductionClass, 'method') as mock_method:
    ...     mock_method.return_value = None
    ...     real = ProductionClass()
    ...     real.method(1, 2, 3)

patch 有其工作的命名空间，只在需要 patch 的地方 patch 是一个要注意的地方。
详情可以阅读 [where to patch][8]

#### MagicMock 和 Mock 的区别

通过查看 mock 的源码，得知
MagicMock 是在一经实例化时，就为其定义了一些常用的方法，方法列表如下：

    magic_methods = (
        "lt le gt ge eq ne "
        "getitem setitem delitem "
        "len contains iter "
        "hash str sizeof "
        "enter exit "
        # we added divmod and rdivmod here instead of numerics
        # because there is no idivmod
        "divmod rdivmod neg pos abs invert "
        "complex int float index "
        "trunc floor ceil "
    )

    numerics = (
        "add sub mul matmul div floordiv mod lshift rshift and xor or pow"
    )
    if inPy3k:
        numerics += ' truediv'
    inplace = ' '.join('i%s' % n for n in numerics.split())
    right = ' '.join('r%s' % n for n in numerics.split())
    extra = ''
    if inPy3k:
        extra = 'bool next '
    else:
        extra = 'unicode long nonzero oct hex truediv rtruediv '

这几个预定义变量 magic_methods，numerics，inplace，right，extra 每个里面的字符
串 s，其前后加 __ 即为最终定义的 magic methods。像 __lt__，__add__,__bool__，
__unicode__ 等，都默认被定义了。

### mox

mox 与 mock 一样也是用来替换 python 对象的，目前的理解，只是实现体验不同。
其与 mock 的差异可以参考：[MoxComparison][9]

其包含两种模式， record 模式和 replay 模式。
在 record 模式创建 Mock，定义 Mock 的方法和行为，；进行 replay 模式并开始使用
Mock，最后使用 verify 来检查 Mock 是否按预期被使用了。

mox 定义了一套截然不同的 API，使用 CreateMock 来创建 Mock，AndReturn 来指定返
回值等。 mox 的文档不是很详细，可以参考一下 [MoxDocumentation][10]。
python 的 mox 模块很小，其实就只有一个文件，可以通读一下，标准的位置一般是在
/usr/lib/python2.7/site-packages/mox.py。

一段示例代码：

    # Create Mox factory
    my_mox = Mox()
    
    # Create a mock data access object
    mock_dao = my_mox.CreateMock(DAOClass)
    
    # Set up expected behavior
    mock_dao.RetrievePersonWithIdentifier('1').AndReturn(person)
    mock_dao.DeletePerson(person)
    
    # Put mocks in replay mode
    my_mox.ReplayAll()
    
    # Inject mock object and run test
    controller.SetDao(mock_dao)
    controller.DeletePersonById('1')
    
    # Verify all methods were called as expected
    my_mox.VerifyAll()

### tox

tox 是用来作持续集成的，它可以协助管理 virtualenv，并在 virtualenv 中进行自动化的测试。

#### 安装 tox

一般都采用 pip 来安装，在基本操作系统上安装 pip (各大发行版的源泉中都有相应的软件包)

centos/fedora: `yum install -y python-pip`
ubuntu/debian: `apt-get install python-pip`

然后使用 pip 来安装 tox

    pip install tox

#### 使用 tox

tox 主要是使用项目中的 tox.ini 来管理，tox.init 的格式参见[tox-config][11]。

这里主要以 openstack nova 项目的 tox.ini 来分析一下：

    [tox]
    minversion = 1.6
    envlist = py34,py27,functional,pep8,pip-missing-reqs
    skipsdist = True

    [testenv]
    usedevelop = True
    # tox is silly... these need to be separated by a newline....
    whitelist_externals = bash
                        find
    install_command = pip install -U --force-reinstall {opts} {packages}
    setenv = VIRTUAL_ENV={envdir}
            OS_TEST_PATH=./nova/tests/unit
            LANGUAGE=en_US
            LC_ALL=en_US.utf-8
    deps = -r{toxinidir}/requirements.txt
        -r{toxinidir}/test-requirements.txt
    commands =
    find . -type f -name "*.pyc" -delete
    bash tools/pretty_tox.sh '{posargs}'
    # there is also secret magic in pretty_tox.sh which lets you run in a fail only
    # mode. To do this define the TRACE_FAILONLY environmental variable.

    [tox:jenkins]
    downloadcache = ~/cache/pip

    [testenv:pep8]
    commands =
    flake8 {posargs}

[tox] 下 envlist 定义了 virtualenv 的列表，然后每个 env 都有其对应的 [testenv:x]
来配置；
[testenv:pep8] 定义了 pep8 这个 virtualenv 下的命令，即是使用 flake8 来检查代码
是否符合规范。
没有明确定义的 py27 则使用默认的 [testenv]：
usedevelop 指定使用 setup.py develop 而非 setup.py sdist，即使用本地代码；
install_command 定义了 virtualenv 中模块的安装方式；
setenv 定义了进入 virtualenv 时的一系列配置命令；
deps 定义了 virtualenv 中需要安装的依赖包，这里依赖包是通过文件的形式写在
requirements.txt 和 test-requirements.txt 下；
commands 定义了执行 `tox -epy27` 时执行的命令，在这里首先删除 *.pyc 文件，然后
执行 tools/pretty_tox.sh 脚本，该脚本的内容即是对 nova 项目进行单元测试。
whitelist_externals 指定了可以使用的 virtualenv 之外的命令。

下面查看一下 pretty_tox.sh 的内容：

    #!/usr/bin/env bash

    set -o pipefail

    TESTRARGS=$1

    # --until-failure is not compatible with --subunit see:
    #
    # https://bugs.launchpad.net/testrepository/+bug/1411804
    #
    # this work around exists until that is addressed
    if [[ "$TESTARGS" =~ "until-failure" ]]; then
        python setup.py testr --slowest --testr-args="$TESTRARGS"
    else
        python setup.py testr --slowest --testr-args="--subunit $TESTRARGS" | subunit-trace -f
    fi

该脚本使用 testrepository(testr) 来自动发现单元测试代码并发起测试。


 [1]: http://testtools.readthedocs.org/en/latest/overview.html
 [2]: https://docs.python.org/2/library/unittest.html
 [3]: http://testtools.readthedocs.org/en/latest/for-test-authors.html#writing-your-own-matchers
 [4]: http://xunitpatterns.com/
 [5]: https://wiki.python.org/moin/PythonTestingToolsTaxonomy
 [6]: https://pypi.python.org/pypi/fixtures
 [7]: https://mock.readthedocs.org/en/latest/
 [8]: https://mock.readthedocs.org/en/latest/patch.html#where-to-patch
 [9]: https://code.google.com/p/pymox/wiki/MoxComparison
 [10]: https://code.google.com/p/pymox/wiki/MoxDocumentation
 [11]: https://tox.readthedocs.org/en/latest/config.html
 [12]: http://testrepository.readthedocs.org/en/latest/MANUAL.html
