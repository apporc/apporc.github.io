routes 模块
==

routes 模块是帮助实现 url 到程序入口映射的工具，解决的问题跟 django 中 urls.py
文件里的相同。这个模块是 ruby 里 routes 模块在 python 中的重新实现。

#### 概览

首先来看一下 routes 工作起来的样子。

    from routes import Mapper
    map = Mapper()
    map.connect(None, "/error/{action}/{id}", controller="error")
    # Match a URL, returns a dict or None if no match
    result = map.match('/error/myapp/4')
    # result == {'controller': 'error', 'action': 'myapp', 'id': '4'}

基本上就是三步，创建一个 Mapper 实例，使用 connect() 注册映射关系，使用 match()
获取一个 url 映射到的结果，如果映射成功返回一个 dict，失败则为 None。
示例中 dict 的各元素值都是字符串，实际的工程中你可以设定它们为你的 class
实例，函数实例等，这样映射出的结果就可以直接指向该请求所应调用的方法，并
同时获得了方法所需的参数。

routes 的 github 位置： [https://github.com/bbangert/routes][1]

#### 注册映射关系

以上的示例中展示了注册映射关系的方式，就是调用 Mapper 的 connect 方法。
在进一步的说明之前，先看一下 routes 的一些术语：

* component:  在一个 url 中被 '/' 分隔开的各个部分
* mapper: 记录了一组映射关系的实例
* matching: 根据一个 url 匹配得到结果的动作
* route:  把一个 url pattern 映射到一个包含 routing variable 字典的规则
* routing variables:  路由变量，就是对一个 Mapper 调用 match() 所得到的结果。

在上面的示例中，connect 方法的第一个参数为这个 route 的名称，第二个参数为 url
pattern，后续的可选参数为额外的路由变量。某一些路由变量是在 url pattern 定义的。

下面列举一下 connect 方法的高级用法。

*   Requirements

    官方文档中叫作 Requirements，其实叫 Constriants 我认为可能更合适一些。
    简单来说，就是我们在注册一个 route 的时候，可以限定 routing variable 需要
    满足的条件。

    存在两种使用方式：

    1.  在 url pattern 中定义

            map.connect(R"/blog/{id:\d+}")
            map.connect(R"/download/{platform:windows|mac}/{filename}")

        第一行限定了 routing variable "id" 必须为数字；第二行限定了 routing
        variable "platform" 必须为 "windows" 或者 "mac" 这两个字符串中的一个。
        开头的 "R" 是为了避免需要跳脱表达式中出现的每一个 '\'。

    2.  指定 requirements 参数

        对应于上面的的例子，如果使用 requirements 参数就是下面这样，本方式的
        好处是，可以把 R"\d+" 这样的限定表达式集中管理，多处使用。

            map.connect("/blog/{id}", requirements={"id": R"\d+"}
            map.connect("/download/{platform}/{filename}",
                requirements={"platform": R"windows|mac"})
            ... ...
            NUMERIC = R"\d+"
            map.connect("/blog/{id}", requirements={"id": NUMERIC}

*   path_info

    这货使用的场合是，使用 routes 只截取 url 中的一部分，剩余的又传递给其它的
    routing 组件。

    看下面这个示例，routes 将 path_info 这个 routing variable 之前的所有内容
    放到 SCRIPT\_NAME 这个环境变量中，把 path\_info 匹配到的内容放到 PATH\_INFO
    环境变量中。

        map.connect(None, "/cards/{path_info:.*}",
            controller="main", action="cards")
        # Incoming URL "/cards/diamonds/4.png"
        => {"controller": "main", action: "cards", "path_info": "/diamonds/4.png"}
        # Second WSGI application sees:
        # SCRIPT_NAME="/cards"   PATH_INFO="/diamonds/4.png"

*   conditions

    conditions 参数也是对于 route 作出限定的工具，只不过它不是用于限定 routing 
    variable，而是用于限定请求的方法以及子域名等。

        # 只匹配 HTTP 方法 "GET" 与 "HEAD"
        m.connect("/user/list", controller="user", action="list",
                conditions=dict(method=["GET", "HEAD"]))

        # 只匹配请求域名为子域名
        m.connect("/", controller="user", action="home",
                conditions=dict(sub_domain=True))

        # 子域名必须为 "fred" 或者 "george" 之一
        m.connect("/", controller="user", action="home",
                conditions=dict(sub_domain=["fred", "george"]))

        # 提供一个函数，用于判定/更改请求中的环境变量信息，当然函数中可以灵活地
        # 做很多事情。
        def referals(environ, result):
            result["referer"] = environ.get("HTTP_REFERER")
            return True
        m.connect("/{controller}/{action}/{id}",
            conditions=dict(function=referals))

*   submapper

    使用 submapper 可以简化 route 规则的注册，便于工程化。

        m = map.submapper(controller="home")
        m.connect("home", "/", action="splash")
        m.connect("index", "/index", action="index")

        with map.submapper(path_prefix="/admin", controller="admin") as m:
            m.connect("admin_users", "/users", action="users")
            m.connect("admin_databases", "/databases", action="databases")

    另外 submapper 默认就有 index, new, create, show, edit, update 和 delete
    这些方法，可供直接使用，如：

        with map.submapper(controller="entries", path_prefix="/entries") as entries:
            entries.index()
            with entries.submapper(path_prefix="/{id}") as entry:
                entry.show()

    也可使用 actions 参数指定

        with map.submapper(controller="entries", path_prefix="/entries",
                        actions=["index"]) as entries:
            entries.submapper(path_prefix="/{id}", actions=["show"])

*   Route

    route 不止是一个术语，对应的也确实有 Route 类可供使用。例如，你可以在不同的
    类中注册其可以处理的 route，然后再在总的 route 类中引用。

        from routes.route import Route
        routes = [
            Route("index", "/index.html", controller="home", action="index"),
            ]

        map.extend(routes)
        # /index.html => {"controller": "home", "action": "index"}

        map.extend(routes, "/subapp")
        # /subapp/index.html => {"controller": "home", "action": "index"}

#### restful

对于 restful api 服务，routes 模块提供了便捷的上层封装，即 resource，
它会为你自动创建大量 restful api 的标准路由规则。

如 `map.resource("message", "messages")` 与下列操作对等：

    map.connect("messages", "/messages",
        controller="messages", action="create",
        conditions=dict(method=["POST"]))
    map.connect("messages", "/messages",
        controller="messages", action="index",
        conditions=dict(method=["GET"]))
    map.connect("formatted_messages", "/messages.{format}",
        controller="messages", action="index",
        conditions=dict(method=["GET"]))
    map.connect("new_message", "/messages/new",
        controller="messages", action="new",
        conditions=dict(method=["GET"]))
    map.connect("formatted_new_message", "/messages/new.{format}",
        controller="messages", action="new",
        conditions=dict(method=["GET"]))
    map.connect("/messages/{id}",
        controller="messages", action="update",
        conditions=dict(method=["PUT"]))
    map.connect("/messages/{id}",
        controller="messages", action="delete",
        conditions=dict(method=["DELETE"]))
    map.connect("edit_message", "/messages/{id}/edit",
        controller="messages", action="edit",
        conditions=dict(method=["GET"]))
    map.connect("formatted_edit_message", "/messages/{id}.{format}/edit",
        controller="messages", action="edit",
        conditions=dict(method=["GET"]))
    map.connect("message", "/messages/{id}",
        controller="messages", action="show",
        conditions=dict(method=["GET"]))
    map.connect("formatted_message", "/messages/{id}.{format}",
        controller="messages", action="show",
        conditions=dict(method=["GET"]))

除此以外，resource 也提供了一些可选参数，便于对 collection 与 member 的
扩充。以下是可选参数的列表

*   controller  用于替换 resource 默认指定的 controller
*   collection  扩展 collection 的操作集

        map.resource("message", "messages", collection={"rss": "GET"})
        # "GET /message/rss"  =>  ``Messages.rss()``.
        # Defines a named route "rss_messages".

*   member  扩展 member 的操作集

        map.resource('message', 'messages', member={'mark':'POST'})
        # "POST /message/1/mark"  =>  ``Messages.mark(1)``
        # also adds named route "mark_message"

另外，还有 path_prefix, name_prefix 和 parent_resource 等可选参数，用于
定义更完善的 url。

#### 生成 url

使用 url_for 帮助函数，可以从一个 url pattern 生成一个 url，便于在需要
一个 url 链接地址的时候随时生成。

    from routes import Mapper
    from routes import url_for
    m = Mapper()
    m.connect("archives", "/archives/{id}",
        controller="archives", action="view", id=1)
    url_for("archives", id=123)  =>  "/archives/123"
    url_for("archives")  =>  "/archives/1"

#### other

其它一些可能会用到小细节

*   匹配 url 后缀

    像传统的 url 格式喜欢在末尾加上一个 ".html" 的情况

        map.connect('/entries/{id}{.format}')
        map.connect('/entries/{id}{.format:json}')

    第一条匹配 “/entries/1” 和 “/entries/1.mp3”；
    第二条匹配 “/entries/1.json”，不匹配 “/entries/1.mp3”。

*   匹配反斜线

    上述介绍 path_info 的时候使用了表达式，而事实上对于任意 routing variable 都是可
    以使用表达式的，类似于 requiremets 中的用法，我们可以把反斜线匹配进来。

        map.connect("/static/{filename:.*?}")

    像 "/static/foo.jpg" 与 "/static/bar/foo.jpg" 就都是可以匹配到的了。

*   Unicode

    route 默认接收到的 url 为 UTF-8 编码，使用 url_for() 函数输出的 url 也采用 UTF-8 编码。

*   Redirect

    使用 Mapper.redirect 可以将一些 url pattern 映射到另外一些 url pattern。

        map.redirect("/legacyapp/archives/{url:.*}", "/archives/{url}")

*   查看 Mapper 已注册的 route

    可以通过直接打印一个 Mapper 实例，也可以遍历其 .matchlist 属性来得到一些
    route 的信息。


routes 的官方文档：[http://routes.readthedocs.io/en/latest/index.html][2]

 [1]: https://github.com/bbangert/routes
 [2]: http://routes.readthedocs.io/en/latest/index.html
