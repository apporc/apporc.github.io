## 概述

python 有大量的单元测试框架，本文尝试对下列模块进行梳理：
testtools，fixture，mox，mock

本文仅从整体的角度对各模块的使用方式进行把握，更详细的信息请随时查阅文中的参考
链接。

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

#### 使用示例

安装 testtools

    pip install testtools

testtools 与 unittest 的使用方式基本是一致的，本文只以 testtools 为例，
下面来看示例代码：

    # -- someserver_test.py --
    from testtools import TestCase
    from myproject import SomeServer

    class SomeTestCase(TestCase):

        def setUp(self):
            super(SomeTestCase， self).setUp()
            self.server = SomeServer()
            self.server.start_up()
            self.addCleanup(self.server.shut_down)

        def test_a_thing(self):
            self.assertEqual("cool"， self.server.temperature)

执行测试：

    python -m testtools.run someserver_test.SomeTestCase

要点：

1.  新建用例继承 testtools.TestCase，如示例中 SomeTestCase。
2.  为 SomeTestCase 定义 setUp 方法，为测试做环境准备。
    示例中生成了待测代码 SomeServer 的一个实例 self.server，并启动它。
    对应的，可以定义 tearDown 方法来进行测试环境的清理。
3.  通过使用 self.addCleanup 定义测试结束或故障退出时如何清理测试环境。
    所添加的函数，会以添加顺序的逆序来执行，该方式比 tearDown 更健壮。
4.  将每一项测试写成名称以 test\_ 开头的方法，执行测试时会执行符合该条件的方法。

#### 断言与调试工具

每一个测试方法，基本上就是包含一系列断言，来判断代码运行的方式。
unittest 所提供的断言方式，testtools 全部都保留了，只是某一些方法会有调整。
类似于 assertTrue()，assertFalse() 等各类断言方法在此就不一一提及了，详细的列表
可以前往 [unittest][2] 和 [testtools][1] 查询。

在此只记录几个特别的用法：

1.  ExpectedException

    结合 with 语句，可以用于判断一断代码会抛出某一种异常

        with testtools.ExpectedException(ZeroDivisionError):
            1/0

2.  expectFailure

    说明预期某一个断言会失败，例如下面 self.assertEqual('cats'， 'dogs') 是预期
    会出错的。

        def test_expect_failure_example(self):
            self.expectFailure(
                "cats should be dogs"， self.assertEqual， 'cats'， 'dogs')

3.  assertThat

    便于自定义断言

        def test_abs(self):
            result = abs(-7)
            self.assertEqual(result， 7)
            self.assertNotEqual(result， 7)
            self.assertThat(result， testtools.matchers.Equals(7))
            self.assertThat(result， testtools.matchers.Not(Equals(8)))

    Equals 或 Not 等 Matchers 可以自定义。 通常就是新建一个类，并为类定义一个 match
    方法，详情参阅 [testtools][3]

4.  Fixtures

    通常当我们需要在不同的测试用例之间进行协调的时候，需要类似下面这样的代码段：

        class SomeTest(TestCase):
            def setUp(self):
                super(SomeTest， self).setUp()
                self.server = Server()
                self.server.setUp()
                self.addCleanup(self.server.tearDown)

    我们会需要调用其它测试用例类的 setUp 函数，并预定何时执行其 tearDown。
    testtools 为此提供了一个更方便的方式 useFixture：

        class SomeTest(TestCase):
            def setUp(self):
                super(SomeTest， self).setUp()
                self.server = self.useFixture(Server())

    这里 Server 必须为符合 [fixture.Fixtures](fixtures) 协议的对象，下文介绍
    fixture 框架。

5.  skipTest

    可以在检测到环境不符合要求的情况下，忽略某些测试

        def test_make_symlink(self):
            symlink = getattr(os， 'symlink'， None)
            if symlink is None:
                self.skipTest("No symlink support")
            symlink(whatever， something_else)

6.  patch

    可以在代码的执行过程中将某一对象进行替换

        def test_foo(self):
            my_stream = StringIO()
            self.patch(sys， 'stderr'， my_stream)
            run_some_code_that_prints_to_stderr()
            self.assertEqual(''， my_stream.getvalue())

    使用自定义的 StringIO 类型 my\_stream 来替换 sys.stderr，便于后续收集
    打印到 stderr 中的信息。

7.  getUniqueString 和 getUniqueInteger

    测试时，在需要使用一些“魔数”和“字符串”的地方，可能通过使用
    getUniqueString 和 getUniqueInteger 来自动生成。

    关于此类方法的详细讨论：[creation methods][4]

#### 运行方式

指定模块或测试用例的方式运行测试

    python -m testtools.run <module.testcase>

指定测试代码的路径，然后自动发现测试用例：

    python -m testtools.run discover <filepath>

### fixtures

前述，测试用例一般都包含一个 setUp 方法来准备测试环境，主要是生成一些测试过程中
会用到的变量与对象。有时候我们会需要在不同的测试用例间共享这些变量与对象，避免
在每一个测试用例中重复定义。所以就有了 fixtures，它制定了一套规范，将测试代码与
环境准备代码分离开来，使得测试代码亦可方便的组织和扩展。

#### 使用示例

安装 fixtures

    pip install fixtures

定义 fixtures

    import fixtures
    class NoddyFixture(fixtures.Fixture):
        def _setUp(self):
            self.frobnozzle = 42
            self.addCleanup(delattr， self， 'frobnozzle')

fixtures 文档中推荐在 \_setUp 方法中定义测试环境，其内建的 setUp 方法会调用
\_setUp，但直接在 setUp 中定义也是允许的。

重用 fixtures

    class ReusingFixture(fixtures.Fixture):
        def _setUp(self):
            self.noddy = self.useFixture(NoddyFixture())
从 unittest.TestCase 或 testtools.TestCase 中使用 fixtures 的时候，需要调用
fixture 对象的 cleanUp 和 setUp 方法。 对于 testtools，其提供了 useFixture
来简化该过程。

    class SomeUnitTest(unittest.TestCase):
        def setUp(self):
            super(SomeTest， self).setUp()
            self.fixture = NoddyFixture()
            self.fixture.setUp()
            self.addCleanup(self.fixture.cleanUp)

    class SomeTesttoolsTest(testtools.TestCase):
        def setUp(self):
            super(SomeTest， self).setUp()
            self.useFixture(NoddyFixture())

fixtures 的详细信息，可以参考 [fixtures][6]

### mock

mock 模块提供的功能是，将 python 对象替换为 Mock 对象，测试中使用到这些对象时，
就会使用相应的 Mock 对象。而这些 Mock 对象会记录下它被怎样的调用了，以备查看。
前述 testtools 提供一个 patch 方法来替换对象， mock 模块也提供了类似方法，此外
mock 模块还负责提供 Mock 对象，这使得你可以使用 mock 来替换任意类型的对象。

python 2.7 及之前版本，需要单独安装 mock 来使用，python 3 之后版本将 mock 集成
进了 unittest，使用 unittest.mock 即可。

mock 模块[详细文档][7]。

#### 代码示例

对于 python 2.7 及之前版本，需安装 mock

    pip install mock

使用 MagicMock 对象来替换对象的方法

    class ProductionClass(object):
        def method(self):
            self.something(1， 2， 3)
        def something(self， a， b， c):
            pass
    real = ProductionClass()
    real.something = MagicMock()
    real.method()
    real.something.assert_called_with(1， 2， 3)
    real.something.assert_called_once_with(1， 2， 3)

使用 Mock 替换普通对象

    class ProductionClass(object):
        def closer(self， something):
            something.close()
    real = ProductionClass()
    mock = Mock()
    real.closer(mock)
    mock.close.assert_called_with()
    # mock 的 close 方法不需要我们定义，当 mock 被调用到 close 时其会自动定义

使用 Mock 替换类

    def some_function():
        instance = module.Foo()
        return instance.method()

    with patch('module.Foo') as mock:
        some_function()
        mock.method.assert_called_with()

查看 Mock 的调用记录

    >>> mock = MagicMock()
    >>> mock.method()
    <MagicMock name='mock.method()' id='...'>
    >>> mock.attribute.method(10， x=53)
    <MagicMock name='mock.attribute.method()' id='...'>
    >>> expected = [call.method()， call.attribute.method(10， x=53)]
    >>> mock.mock_calls
    [call.method()， call.attribute.method(10， x=53)]
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

使用 Mock 的 side\_effect

    # 当需要 Mock 或其方法每次的调用返回值不同时，使用 side_effect
    >>> vals = {(1， 2): 1， (2， 3): 2}
    >>> def side_effect(*args):
    ...     return vals[args]
    ...
    >>> mock = MagicMock(side_effect=side_effect)
    >>> mock(1， 2)
    1
    >>> mock(2， 3)
    2

根据已有类创建 Mock，只有类中已有的方法会在 Mock 中出现

    # 传递 spec 给 Mock，SomeClass 所没有的方法在 Mock 中将不会存在。
    mock = Mock(spec=SomeClass)
    mock.some_unknown_method()
    # 调用不存在的方法会抛出 AttributeError

#### 使用 mock 的 patch 模块

使用 patch.object 方法将对象的方法替换为自定义方法，使用 patch 方法来替换模块，
既可以使用修饰符，也可以使用 with 语句。

    >>> class MyTest(unittest2.TestCase):
    ...     # 使用自定义方法来替换原有方法
    ...     @patch.object(SomeClass， 'attribute'， sentinel.attribute)
    ...     # 使用自动的 Mock 对象来替换原有对象，Mock 对象会被作为参数自动传给
    ...     # 测试函数，注意顺序与修饰时顺序相反。
    ...     @patch('package.module.ClassName1')
    ...     @patch('package.module.ClassName2')
    ...     def test_something(self， MockClass2， MockClass1):
    ...         self.assertEqual(SomeClass.attribute， sentinel.attribute)
    ...         self.assertTrue(package.module.ClassName1 is MockClass1)
    ...         self.assertTrue(package.module.ClassName2 is MockClass2)
    ...
    >>> original = SomeClass.attribute
    >>> MyTest('test_something').test_something()
    # 在 patch 所修饰的函数之外，替换没有作用，patch 有其作用域
    >>> assert SomeClass.attribute == original
    >>> class ProductionClass(object):
    ...     def method(self):
    ...         pass
    ...
    >>> with patch.object(ProductionClass， 'method') as mock_method:
    ...     mock_method.return_value = None
    ...     real = ProductionClass()
    ...     real.method(1， 2， 3)

patch 有其作用域，提倡只在需要 patch 的地方进行 patch，对此可以参考
[where to patch][8]。

#### MagicMock 和 Mock 的区别

通过查看 mock 的源码，得知 MagicMock 是在一经实例化时，就为其定义了一
些常用的方法，方法列表如下：

    "lt le gt ge eq ne "
    "getitem setitem delitem "
    "len contains iter "
    "hash str sizeof "
    "enter exit "
    "divmod rdivmod neg pos abs invert "
    "complex int float index "
    "trunc floor ceil "
    "add sub mul matmul div floordiv mod lshift rshift and xor or pow"
    'bool next '
    'unicode long nonzero oct hex truediv rtruediv '

以上字符串 s，其前后加 \_\_ 即为最终定义的方法 。像 \_\_lt\_\_，\_\_add\_\_，
\_\_bool\_\_，\_\_unicode\_\_ 等，都默认被定义了。

### mox

mox 与 mock 一样也是用来替换 python 对象的，目前的理解，只是实现体验不同。
其与 mock 的差异可以参考：[MoxComparison][9]

其包含两种模式， record 模式和 replay 模式。
在 record 模式创建 Mock，定义 Mock 的方法和行为，；进入 replay 模式并开始使用
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
