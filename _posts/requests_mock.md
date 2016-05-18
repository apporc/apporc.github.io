requests-mock 模块
==
#### 背景

微服务盛行的当下，服务之间的交互变得极为频繁，这带来了更好的服务模块化，
但也为开发和调试带来了一个问题，即当你正专注于解决服务 A 的问题的时候，
却常常被它的依赖服务 B 的问题搞得焦头烂额。

requests-mock 这个模块就是解决此类问题的，它通过在服务 A 中模拟所依赖的
服务 B 的 api 接口，而为服务 A 提供一致稳定的响应，从而隔离掉服务 B
所可能带来的问题。

模块地址：[https://pypi.python.org/pypi/requests-mock][1]

#### 使用 Mocker

下面来实作一下，看几个官方的例子

*   以 context manager 的形式使用 Mocker

        >>> import requests
        >>> import requests_mock

        >>> with requests_mock.Mocker() as m:
        ...     m.get('http://test.com', text='resp')
        ...     requests.get('http://test.com').text
        ...
        'resp'

*   以 function decorator 的形式使用 Mocker

        >>> @requests_mock.Mocker()
        ... def test_function(m):
        ...     m.get('http://test.com', text='resp')
        ...     return requests.get('http://test.com').text
        ...
        >>> test_function()
        'resp'

*   以 class decorator 的形式使用 Mocker

        >>> requests_mock.Mocker.TEST_PREFIX = 'foo'
        >>>
        >>> @requests_mock.Mocker()
        ... class Thing(object):
        ...     def foo_one(self, m):
        ...        m.register_uri('GET', 'http://test.com', text='resp')
        ...        return requests.get('http://test.com').text
        ...     def foo_two(self, m):
        ...         m.register_uri('GET', 'http://test.com', text='resp')
        ...         return requests.get('http://test.com').text
        ...

综上，Mocker 是 requests-mock 提供的一个上层抽象，其定义了一系列与 HTTP
方法同名的方法，如 get, put, post 等。通过调用这些方法，就相当于是注册了
对相应 HTTP 方法的模拟，示例中使用 Mocker().get("http://test.com", text="resp")
的形式模拟了 http://test.com 的 GET 方法，指定返回值为 resp；  
另外，如果你不想使用 Mocker 的这一些封装后的高级方法，你也可以使用
register\_uri 方法来自行定义；  
Mocker 可以作为 context manager 或者 decorator 来使用，这些都方便了在
单元测试中快速的整合，就像有一些测试框架只把单元测试类中以 test\_ 开头的方法
识别为测试用例入口一样， requests-mock 也支持设定类的方法前缀，在上面
第三个示例中，指定 Mocker.TEST_PREFIX 为 foo 就是这个目的。

注意：

实例化一个 Mocker 实例，就意味着当前的进程空间中的 requests 模块被动了
手脚，过去发往真实外部服务的请求会被拦截从而拿到我们模拟的结果，而如果一
个请求我们没有进行模拟却发起了访问，这种情况下会抛出 NoMockAddress 异常；
对于我们未模拟的请求，我们也可以让它访问真实的服务地址，方法是在实例化
Mocker 的时候指定 real_http 为 True，如下：

    Mocker(real_http=True)

#### 使用 Adapter

由上面的示例不难猜到 Mocker.get(), Mocker.post() 等方法背后实际使用了
Mocker.register_uri() 方法来达成目的。
那么 Mocker.register_uri() 之类的方法背后又是使用了什么呢。

这就要提到 Mocker 背后的 Adapter 实例，如同所有的框架一样，在提供了上
层抽象之外，你也可以使用框架自身的底层抽象来实现更为灵活的功能。

下面是官方文档中直接使用 Adapter 的一些示例：

*   实例化 Adapter，并同时对 session 作手脚

        >>> import requests
        >>> import requests_mock
        >>> adapter = requests_mock.Adapter()
        >>> session = requests.Session()
        >>> session.mount('mock', adapter)

    示例中初始化了 Adapter()，然后通过 session.mount('mock', adapter)
    的形式提示 requests 模块把发往 'mock' 协议的请求拦截至 adapter，
    实际测试中你就用 http 或者 https 都好啦，由于 requests.session
    的工作机制就是对于不再的协议使用了一些自定义的 adapter 来处理，
    这里完全借用 requests 自身的机制。

*   匹配请求

    Adapter 也有自己的 register_uri 方法，所以怎么用，你懂得

        .. >>> adapter.register_uri('GET', 'mock://test.com/path', text='resp')
        .. >>> session.get('mock://test.com/path').text
        .. 'resp'

    有必要看的是匹配请求的一些细节。

    *   匹配域名

            .. >>> adapter.register_uri('GET', '//test.com/', text='resp')
            .. >>> session.get('mock://test.com/').text
            .. 'resp'
    *   匹配路径

            .. >>> adapter.register_uri('GET', '/path', text='resp')
            .. >>> session.get('mock://test.com/path').text
            .. 'resp'
            .. >>> session.get('mock://another.com/path').text
            .. 'resp'

    *   匹配请求字符串

            >>> adapter.register_uri('GET', '/7?a=1', text='resp')
            >>> session.get('mock://test.com/7?a=1&b=2').text
            'resp'

    *   匹配所有任意请求

            >>> adapter.register_uri(requests_mock.ANY, requests_mock.ANY, text='resp')
            >>> session.get('mock://whatever/you/like').text
            'resp'
            >>> session.post('mock://whatever/you/like').text
            'resp'

    *   匹配正则表达式

            .. >>> import re
            .. >>> matcher = re.compile('tester.com/a')
            .. >>> adapter.register_uri('GET', matcher, text='resp')
            .. >>> session.get('mock://www.tester.com/a/b').text
            .. 'resp'

    *   匹配 headers

            >>> adapter.register_uri('POST', 'mock://test.com/headers', request_headers={'key': 'val'}, text='resp')
            >>> session.post('mock://test.com/headers', headers={'key': 'val', 'another': 'header'}).text
            'resp'

#### 自定义匹配

你或许要说以上都是 bullshit，这样零零散散怎么工程化，总不能就扔无数行
register_uri 放那吧。那我们看一下 Adapter 的自定义匹配。

    >>> def custom_matcher(request):
    ...     if request.path_url == '/test':
    ...         resp = requests.Response()
    ...         resp.status_code = 200
    ...         return resp
    ...     return None
    ...
    >>> adapter.add_matcher(custom_matcher)
    >>> session.get('mock://test.com/test').status_code
    200

好啦，如此可以自由匹配了。但是还有一个问题，上面提到的那个
`session.mount('mock', adapter)` 很烦恼啊，每次都只能匹配一种协议昂，而且
是限定在一个唯一 session 下的啊。这个问题，官方文档里也没有结出答案呃，
它就让你去用 Mocker 的方式了。不过我查到其实掩藏在上面那一堆
context manager，decorator 后面的机制其实是， Mocker 有一个 start 方法
来开启 requests 请求的拦截，而且它比 session.mount() 要来得彻底多了。
且看代码：

    class Mocker(object):
        def start(self):
            ... ...
            self._real_send = requests.Session.send

            def _fake_get_adapter(session, url):
                return self._adapter

            def _fake_send(session, request, **kwargs):
                real_get_adapter = requests.Session.get_adapter
                requests.Session.get_adapter = _fake_get_adapter
                ... ...
                return self._real_send(session, request, **kwargs)

            requests.Session.send = _fake_send
这里可谓真得是对 requests 作了手术啊，直接把 Session 类的 send 方法
给替换了，这样就不管你是什么协议，最终都被拦截到这个 \_fake\_send
方法中了。而且我们也可以看到 Mocker 有一个自己的 Adapter 实例，就
是 self.\_adapter。最激动人心的是，Adapter 身上的方法也可以直接用
在 Mocker 实例身上啊。因为 Mocker 有这么一方法：

    class Mocker(object):

        def __getattr__(self, name):
            if name in self._PROXY_FUNCS:
                try:
                    return getattr(self._adapter, name)
                except AttributeError:
                    pass

\_PROXY\_FUNCS 即暴露到 Mocker 身上的 Adapter 方法是这样的：

        _PROXY_FUNCS = set(['last_request',
                            'register_uri',
                            'add_matcher',
                            'request_history',
                            'called',
                            'call_count'])


好啦，结合以上种种，下面这种使用方法就出锅了:
    
    import requests
    from requests_mock import mock
    request_mock = mock()

    class FakeBackend(object):

        def __call__(self, request):
            if request.path_url == '/test':
                resp = requests.Response()
                resp.status_code = 200
                return resp
            return None

    fake_backend = Fake_backend()
    request_mock.start()
    request_mock.add_matcher(fake_backend)

注意： mock 与 Mocker 同，在 requests_mock 中有 "mock=Mocker"

#### 使用 Response

上面已经看到一些构建假 response 的方法了，这里稍微介绍一下细节

*   对于使用 register_uri 的方式

    除了可以指定 text 以外，还有这些参数可供指定：

        status_code:	The HTTP status response to return. Defaults to 200.
        reason:	The reason text that accompanies the Status (e.g. ‘OK’ in ‘200 OK’)
        headers:	A dictionary of headers to be included in the response.
        json:	A python object that will be converted to a JSON string.
        text:	A unicode string. This is typically what you will want to use for regular textual content.
        content:	A byte string. This should be used for including binary data in responses.
        body:	A file like object that contains a .read() function.
        raw:	A prepopulated urllib3.response.HTTPResponse to be returned.

    来个示例：

        >>> adapter.register_uri('GET', 'mock://test.com/1', json={'a': 'b'}, status_code=200)
        >>> resp = session.get('mock://test.com/1')
        >>> resp.json()
        {'a': 'b'}

    对于 text, json, context, body, raw 等 body 参数，也可以指定一个回调函数，示例：

        >>> def text_callback(request, context):
        ...     context.status_code = 200
        ...     context.headers['Test1'] = 'value1'
        ...     return 'response'
        ...
        >>> adapter.register_uri('GET',
        ...                      'mock://test.com/3',
        ...                      text=text_callback,
        ...                      headers={'Test2': 'value2'},
        ...                      status_code=400)
        >>> resp = session.get('mock://test.com/3')
        >>> resp.status_code, resp.headers, resp.text
        (200, {'Test1': 'value1', 'Test2': 'value2'}, 'response')

    回调函数的参数除了 request 之外，还有一个 context，context 的属性有：

        headers:	The dictionary of headers that are to be returned in the response.
        status_code:	The status code that is to be returned in the response.
        reason:	The string HTTP status code reason that is to be returned in the response.

#### 请求历史

下面这个是单元测试必备的，就是一个 Mocker 是否被请求，以及曾经接到过哪些请求

示例：

    >>> import requests
    >>> import requests_mock

    >>> with requests_mock.mock() as m:
    ...     m.get('http://test.com, text='resp')
    ...     resp = requests.get('http://test.com')
    ...
    >>> m.called
    True
    >>> m.call_count
    1
    >>> history = m.request_history
    >>> len(history)
    1
    >>> history[0].method
    'GET'
    >>> history[0].url
    'http://test.com/'

#### Fixtures

最后这个，是与单元测试框架中的 fixtures 相关的，fixtures 在本文内容之外了，
所以就不作介绍。熟悉单元测试的人，看到下面这个示例就懂了：

    >>> import requests
    >>> from requests_mock.contrib import fixture
    >>> import testtools

    >>> class MyTestCase(testtools.TestCase):
    ...
    ...     TEST_URL = 'http://www.google.com'
    ...
    ...     def setUp(self):
    ...         super(MyTestCase, self).setUp()
    ...         self.requests_mock = self.useFixture(requests_mock.Mock())
    ...         self.requests_mock.register_uri('GET', self.TEST_URL, text='respA')
    ...
    ...     def test_method(self):
    ...         self.requests_mock.register_uri('POST', self.TEST_URL, text='respB')
    ...         resp = requests.get(self.TEST_URL)
    ...         self.assertEqual('respA', resp.text)
    ...         self.assertEqual(self.TEST_URL, self.requests_mock.last_request.url)
    ...

官方文档链接在此：[http://requests-mock.readthedocs.io/en/latest/overview.html][2]

 [1]: https://pypi.python.org/pypi/requests-mock
 [2]: http://requests-mock.readthedocs.io/en/latest/overview.html
