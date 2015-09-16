---
layout: post
title:  "Openstack 核心框架：eventlet"
date:   2015-09-16 10:20:00
categories: OpenStack
---
1.  简介

    eventlet 是一个 python 并发网络库，它采用 epoll/pool/select等异步 IO，并结合
    greenlet 协程库，使得可以以串行编码的方式编写异步并发代码。
    在目前的 openstack 版本中仍然是驱动整个架构的基石，未来会[被 asyncio 替换][7]。

    理解 eventlet 需要用到一些概念：

    *   并发与并行
    关于并发与并行的区别，Sun 的[多线程编程指南][1]说得比较精辟：
    并发是两个以上的线程同时取得进展的情况，并行是两个以上的线程同时在执行指令的情况，并发不
    一定并行。或者有时间也可以浏览一下这个幻灯片：[并发不是并行][2]。

    *   异步与同步
    简化来说，同步指当进行一项任务的时候只有等待其执行完毕才可以切换到下一个任务，而异步则指
    不必等待而直接切换至下一个任务。在 [stackoverflow][4] 上有一个信息非常丰富的页面。

    *   阻塞与非阻塞
    另外一个需要留意的概念是 blocking IO 和 non-blocking IO，这两个概念与同步/异步概念一起来
    看，当进行阻塞式 IO 的时候，当前任务会被卡住，无法切换到下一个任务。详细情况可以
    参考[维基百科][3]。

    *   协程
        操作系统中的执行实体，由大到小依次是进程，线程和协程。协程是最为轻便的，同一个线程中
        可以运行多个协程。更详细的情况见[协程][6]

    粗略来说，本文关注的点即是：eventlet 是帮助实现并发的 python 库，不一定并行，但是可以并
    行；在讲究高并发与分布式的今天，同步和阻塞就慢，异步和非阻塞就快。

2.  GreenThread

    eventlet 的 GreenThread 封装了 greenlet，首先看一下 greenlet。

    greenlet 是从 [stackless][8] 中拆分出来的项目，其为 python 所带来的功能加成就是"绿色线程"
    ，或者也叫"微线程"，"协程"等，现在 Go 语言中也有类似的实现，名字又唤作"例程"。
    协程是一种运行在线程中的执行单位，同一个线程中可以同时运行多个协程，注意：该同时运行并不
    是真正意义上的并行，同时拥有 CPU 时间片的只有一个当前协程，其它的协程处于挂起状态。不要觉
    得这样"协程"就没有意义，因为有另一类协程，当前需要的不是 CPU 而是 IO。众所周知，CPU 是很
    快的，而 IO 很慢，所以当一些协程都在挂起等待 IO 的时候，一些协程则在用 CPU，在同一个线程
    中跑多个协程的就可以更充分地利用硬件资源，极大的提高服务的响应能力。
    不同于线程和进程由内核来调度并分配时间片，协程需要在代码层显式地控制与转换。

    来看一个官网上的代码示例：

        from greenlet import greenlet

        def test1():
            print 12
            gr2.switch()
            print 34

        def test2():
            print 56
            gr1.switch()
            print 78

        gr1 = greenlet(test1)
        gr2 = greenlet(test2)
        gr1.switch()
        
    示例创建了两个协程 gr1 和 gr2，分别运行 test1 与 test2 两个函数，协程之间的切换使用
    switch 方法。示例一开始从创建这两个协程的主协程，切换到 gr1 去执行，并打印 12，之后
    切换到 gr2，打印 56，再切换回 gr1 打印 34，gr1 执行完成，执行权限重回主协程，由于之后
    主协程没有再切换给 gr2 执行，所以 78 没有被打印。

    greenlet 是非常精简的库，可以参考这里 [greenlet][9]

    eventlet 将 greenlet 封装为一个 GreenThread 类：

        class GreenThread(greenlet.greenlet):
            """The GreenThread class is a type of Greenlet which has the additional
            property of being able to retrieve the return value of the main function.
            Do not construct GreenThread objects directly; call :func:`spawn` to get one.
            """

            def __init__(self, parent):
                greenlet.greenlet.__init__(self, self.main, parent)
                self._exit_event = event.Event()
                self._resolving_links = False

3.  EDSM

    关于互联网应用的一些常用架构，参考[这里][5]。除了多进程与多线程架构之外，在面对巨大请求
    压力的时候，EDSM 是必须要采用的一种架构。其核心价值是使需要等待 IO 的进线程不必阻塞并继
    续占用线程执行空间，让线程得到高效利用。这种框架一般是采用 epool，pool，select 等来等待
    网络 IO 事件，程序逻辑为事件添加回调函数。当网络事件发生时，通过调用回调函数来处理请求。
    通常回调函数所注册到的这个抽象叫作 event loop。 [参考 asyncio and python][10] 和 [C10K][11]

    传统的 EDSM 都是依赖于回调函数链来处理请求，使用回调函数链写成的代码不直观，难于维护。
    回调函数链复杂起来非常恐怖，所以有 "Callback Hell" 的说法。eventlet 的好处就在于，其使用
    greenlet 封装出了不必使用回调函数链的方法。使用 eventlet 写代码的时候，就像平常写串行代码
    一样，而代码最终却是并发执行的。

    evenlet 的 event loop 叫作 hub。hub 支持以上提到的 epool，pool，select 还支持 libevent。
    直观来说，eventlet 就是一个 hub 之下挂着多个协程，每个协程在遇到需要 IO 或着 sleep 操作时
    都往 hub 注册一个事件并把自己挂起，当 hub 检测到相应的事件后，再将协程唤醒。

    eventlet 提供了 hub 选择的接口：

        eventlet.hubs.use_hub(hub=None)

    eventlet 的 Hub() 是一层对 epool，pool 等的封装，实现了添加文件描述符与回调函数对的接口，
    以及添加定时器等。

        class BaseHub(object):
            """ Base hub class for easing the implementation of subclasses that are
            specific to a particular underlying event architecture. """

        def __init__(self, clock=time.time):
            self.listeners = {READ: {}, WRITE: {}}
            self.clock = clock
            self.greenlet = greenlet.greenlet(self.run)
            self.stopping = False
            self.running = False
            self.timers = []
            self.next_timers = []
            self.lclass = FdListener
            ... ...

    Hub() 维护了等待 IO 事件的文件描述符列表在 self.listeners 中；
    self.greenlet 为一个 Hub() 的主协程，它从创建起即运行 Hub().run()。

        def add(self, evtype, fileno, cb, tb, mark_as_closed):

            listener = self.lclass(evtype, fileno, cb, tb, mark_as_closed)
            ... ...
            return listener

    add() 方法是为 Hub() 添加文件描述符的接口。

        def add_timer(self, timer):
            scheduled_time = self.clock() + timer.seconds
            self.next_timers.append((scheduled_time, timer))

    add_timer() 添加定时器，假如在协程中调用 greenthread.sleep()，相应协程就会依此在
    Hub 中添加定时器，让 Hub 在时间到来时将其唤醒。

        def wait(self, seconds=None):
            readers = self.listeners[READ]
            writers = self.listeners[WRITE]

            if not readers and not writers:
                if seconds:
                    sleep(seconds)
                return
            try:
                presult = self.do_poll(seconds)
            except (IOError, select.error) as e:
                if get_errno(e) == errno.EINTR:
                    return
                raise
            SYSTEM_EXCEPTIONS = self.SYSTEM_EXCEPTIONS

            # Accumulate the listeners to call back to prior to
            # triggering any of them. This is to keep the set
            # of callbacks in sync with the events we've just
            # polled for. It prevents one handler from invalidating
            # another.
            callbacks = set()
            for fileno, event in presult:
                if event & READ_MASK:
                    callbacks.add((readers.get(fileno, noop), fileno))
                if event & WRITE_MASK:
                    callbacks.add((writers.get(fileno, noop), fileno))
                if event & select.POLLNVAL:
                    self.remove_descriptor(fileno)
                    continue
                if event & EXC_MASK:
                    callbacks.add((readers.get(fileno, noop), fileno))
                    callbacks.add((writers.get(fileno, noop), fileno))

            for listener, fileno in callbacks:
                try:
                    listener.cb(fileno)
                except SYSTEM_EXCEPTIONS:
                    raise
                except:
                    self.squelch_exception(fileno, sys.exc_info())
                    clear_sys_exc_info()

    wait() 方法对文件描述符列表调用 poll 操作来监听 IO 事件，事件来到时，调用相应注册的
    回调函数，整个线程就会切换到注册回调函数的那个协程中去执行。

        def prepare_timers(self):
            heappush = heapq.heappush
            t = self.timers
            for item in self.next_timers:
                if item[1].called:
                    self.timers_canceled -= 1
                else:
                    heappush(t, item)
            del self.next_timers[:]

        def sleep_until(self):
            t = self.timers
            if not t:
                return None
            return t[0][0]

        def run(self, *a, **kw):
            """Run the runloop until abort is called.
            """
            # accept and discard variable arguments because they will be
            # supplied if other greenlets have run and exited before the
            # hub's greenlet gets a chance to run
            if self.running:
                raise RuntimeError("Already running!")
            try:
                self.running = True
                self.stopping = False
                while not self.stopping:
                    while self.closed:
                        # We ditch all of these first.
                        self.close_one()
                    self.prepare_timers()
                    if self.debug_blocking:
                        self.block_detect_pre()
                    self.fire_timers(self.clock())
                    if self.debug_blocking:
                        self.block_detect_post()
                    self.prepare_timers()
                    wakeup_when = self.sleep_until()
                    if wakeup_when is None:
                        sleep_time = self.default_sleep()
                    else:
                        sleep_time = wakeup_when - self.clock()
                    if sleep_time > 0:
                        self.wait(sleep_time)
                    else:
                        self.wait(0)
                else:
                    self.timers_canceled = 0
                    del self.timers[:]
                    del self.next_timers[:]
            finally:
                self.running = False
                self.stopping = False

    run() 方法是 Hub 的执行入口，主协程 self.greenlet 执行该方法。方法中是一个永久执行的循环
    ，每次循环体都会处理完定时器后调用 wait() 来对各文件描述符进行 poll 操作。
    处理定时器的操作是遍历所有的定时器，如果已经到时间则开始执行其对应的回调函数，也就是唤醒
    协程来执行。之后从剩余定时器中取出第一个出来，在它的时间之内，调用 wait() 去等待 IO 事件
    ，时间超出之后又从头执行 run() 方法。


    初始化的时候，会根据所选的工具进行 poll 操作，下面是 epolls.py 的实现。

        class Hub(poll.Hub):
            def __init__(self, clock=time.time):
                BaseHub.__init__(self, clock)
                self.poll = epoll()

        def do_poll(self, seconds):
            return self.poll.poll(seconds)

4.  Non-blocking IO

    以上说了 Hub() 是如何在监听到 IO 事件或定时器到点之后唤醒协程的，与之相对应的，协程在需要
    等待 IO 事件或者进行 sleep() 时，则需要将自身挂起，并切换到主协程。
    这就需要使用非阻塞式的 IO，而 python 标准库中的模块都是阻塞式的 IO。eventlet
    为这些每一个模块都写了替换模块，以使用异步 IO。

    有两种方式来使用这些非阻塞式的模块。

    第一种是直接引用与标准模块同名的 eventlet 版本的模块，例如：

        from eventlet.green import socket
        from eventlet.green import threading

    第二种是对标准模块进行运行时替换(通过可选参数有选择性的只处理某些模块)：

        import eventlet
        eventlet.monkey_patch(os=None, select=None, socket=None, thread=None, time=None, psycopg=None)

    这些被替换的模块，所要完成的任务就是，在正常的 IO 操作之时为 socket 设置 non-blocking，
    之后并通过如下函数将文件描述符注册到 Hub，注册到 Hub 同时即将当前作 IO 操作的协程挂起，
    并将主协程唤醒：

        def trampoline(fd, read=None, write=None, timeout=None,
                    timeout_exc=timeout.Timeout,
                    mark_as_closed=None):
            t = None
            hub = get_hub()
            current = greenlet.getcurrent()
            assert hub.greenlet is not current, 'do not call blocking functions from the mainloop'
            assert not (
                read and write), 'not allowed to trampoline for reading and writing'
            try:
                fileno = fd.fileno()
            except AttributeError:
                fileno = fd
            if timeout is not None:
                def _timeout(exc):
                    # This is only useful to insert debugging
                    current.throw(exc)
                t = hub.schedule_call_global(timeout, _timeout, timeout_exc)
            try:
                if read:
                    listener = hub.add(hub.READ, fileno, current.switch, current.throw, mark_as_closed)
                elif write:
                    listener = hub.add(hub.WRITE, fileno, current.switch, current.throw, mark_as_closed)
                try:
                    return hub.switch()
                finally:
                    hub.remove(listener)
            finally:
                if t is not None:
                    t.cancel()

    以下以 os 模块的 read 方法为例：

        def read(fd, n):

            while True:
                try:
                    return __original_read__(fd, n)
                except (OSError, IOError) as e:
                    if get_errno(e) != errno.EAGAIN:
                        raise
                except socket.error as e:
                    if get_errno(e) == errno.EPIPE:
                        return ''
                    raise
                try:
                    hubs.trampoline(fd, read=True)
                except hubs.IOClosed:
                    return ''

    __original_read__ 是 os 模块原版的 read 方法，在这里调用了原版的 read 方法之后，
    将调用了 trampoline 来把文件描述符 fd 注册到了 Hub 中。


    关于 monkey_patch 方法是如何在运行时将标准库模块进行替换，代码如下：

        if on['thread'] and not already_patched.get('thread'):
            modules_to_patch += _green_thread_modules()
            already_patched['thread'] = True

        def _green_thread_modules():
            from eventlet.green import Queue
            from eventlet.green import thread
            from eventlet.green import threading
            if six.PY2:
                return [('Queue', Queue), ('thread', thread), ('threading', threading)]
            if six.PY3:
                return [('queue', Queue), ('_thread', thread), ('threading', threading)]

        for name, mod in modules_to_patch:
            orig_mod = sys.modules.get(name)
            if orig_mod is None:
                orig_mod = __import__(name)
            for attr_name in mod.__patched__:
                patched_attr = getattr(mod, attr_name, None)
                if patched_attr is not None:
                    setattr(orig_mod, attr_name, patched_attr)

    以 os 模块为例，其是将标准库 os 模块的一些属性在运行时替换为 eventlet 版本 os
    模块的相应属性，相应属性如上所示已经被 eventlet 加入了 trampoline 的调用：

        __patched__ = ['fdopen', 'read', 'write', 'wait', 'waitpid', 'open']


5.  本地线程

    由于阻塞式的任务不能在 eventlet 协程中运行，然而除了标准库的模块之外，现实世界中也经常会
    用到第三方的阻塞式的库。对于这类的库， eventlet 也有一种办法，就是将会阻塞的操作放到本
    地线程中去运行。
    eventlet 为此也作了一个封装接口。在 tpool 模块之中。

        import thread
        from eventlet import tpool
        def my_func(starting_ident):
            print("running in new thread:", starting_ident != thread.get_ident())
        tpool.execute(my_func, thread.get_ident)

    除了 tpool.execute 之外，还有一个 tpool.Proxy(obj, autowrap=(), autowrap_names=()) 的用法，
    在 openstack 中普通使用。该方法接受一个实例，实例是某阻塞式对象的代理，将调用转发给相应对
    象。
    需要注意的是，代理对象中不能用协程切换相关的操作。因为操作已经是另一个本地线程之中了。

    这个的用法可以参考 oslo.db 中一段对 sqlalchemy 的处理。

        class TpoolDbapiWrapper(object):

            @property
            def _api(self):
                if not self._db_api:
                    with self._lock:
                        if not self._db_api:
                            db_api = api.DBAPI.from_config(
                                conf=self._conf, backend_mapping=self._backend_mapping)
                            if self._conf.database.use_tpool:
                                try:
                                    from eventlet import tpool
                                except ImportError:
                                    LOG.exception(_LE("'eventlet' is required for "
                                                    "TpoolDbapiWrapper."))
                                    raise
                                self._db_api = tpool.Proxy(db_api)
                            else:
                                self._db_api = db_api
                return self._db_api



        class DBAPI(object):

            def _load_backend(self):
                with self._lock:
                    if not self._backend:
                        # Import the untranslated name if we don't have a mapping
                        backend_path = self._backend_mapping.get(self._backend_name,
                                                                self._backend_name)
                        backend_mod = importutils.try_import(backend_path)
                        if not backend_mod:
                            raise ImportError("Unable to import backend '%s'" %
                                            self._backend_name)
                        self._backend = backend_mod.get_backend()

            def __getattr__(self, key):
                if not self._backend:
                    self._load_backend()

                attr = getattr(self._backend, key)
                if not hasattr(attr, '__call__'):
                    return attr

    __getattr__ 将对代理对象的调用都转发到了其背后需要封装的对象。

6.  eventlet 在 openstack 中的使用

    openstack 中所使用的方法是 monkey_patch 的方法，通常在一个进/线程开始的时候对所
    有的标准模块进行替换。例如， nova 的做法是在 nova/cmd/__init__.py 中进行：

        import eventlet
        eventlet.monkey_patch()

    由于所有的 nova 服务，如 nova-api, nova-scheduler, nova-compute 的执行入口都在
    nova/cmd/ 下，所以当各服务开始执行的时候都做了模块的替换。

    通常发起协程进行任务的流程如下：

        def mission():
            print('working')

        pool_size = 10
        pool = eventlet.greenpool.GreenPool(pool_size)
        pool.spawn(mission)

    设定 pool 中的协程数目，通过 spawn 让 pool 把任务扔给协程。

    具体到将任务放进 GreenThread 中执行，以下是 oslo.messaging 中从收到 rpc 信息到
    发起任务的过程：

        def spawn_with(ctxt, pool):
            def complete(thread, exit):
                exc = True
                try:
                    try:
                        thread.wait()
                    except Exception:
                        exc = False
                        if not exit(*sys.exc_info()):
                            raise
                finally:
                    if exc:
                        exit(None, None, None)

            callback = ctxt.__enter__()
            thread = pool.spawn(callback)
            thread.link(complete, ctxt.__exit__)

            return thread
    ctxt 是为一个消息匹配到的处理逻辑，pool 是 GreenPool，在这里为生成的协程通过
    link 注册了一个回调，处理一些残余信息。

        class EventletExecutor(base.ExecutorBase):

            def __init__(self, conf, listener, dispatcher):
                super(EventletExecutor, self).__init__(conf, listener, dispatcher)
                self.conf.register_opts(_eventlet_opts)
                self._thread = None
                self._greenpool = greenpool.GreenPool(self.conf.rpc_thread_pool_size)
                self._running = False

            def start(self):
                if self._thread is not None:
                    return

                @excutils.forever_retry_uncaught_exceptions
                def _executor_thread():
                    try:
                        while self._running:
                            incoming = self.listener.poll()
                            if incoming is not None:
                                spawn_with(ctxt=self.dispatcher(incoming),
                                        pool=self._greenpool)
                    except greenlet.GreenletExit:
                        return

                self._running = True
                self._thread = eventlet.spawn(_executor_thread)

    每个 EventletExecutor 维护了一个 GreenPool，一个 GreenPool 维护了多个 GreenThread，
    当收到 rpc 消息时，在一个 GreenThread 中执行 self.dispatcher 对消息进行解析，并匹配
    到相应的处理逻辑中去。 _executor_thread 则循环收取 rpc 消息，分发任务。

7.  缺点

    eventlet 所提供的舒适的串行编程风格，带来一些好处的同时，也有一些弊端。

    首先，不得不维护项目中所有的模块，保证其没有阻塞式的调用。在使用众多第三方库的的大型
    项目中这个也是比较复杂的。

    其次，因为在协程之间的切换是隐式的，如果出现了协程间的争用时，在代码中理清协程何时进
    行切换也是不容易的。

    另外还有对 pypy 和 python3 支持的问题。详情可以参考[asyncio blueprint][7] 和
    [unyielding][12]

8.  参考链接

[http://docs.oracle.com/cd/E19455-01/806-5257/6je9h032b/index.html][1]
[http://concur.rspace.googlecode.com/hg/talk/concur.html#landing-slide][2]
[https://en.wikipedia.org/wiki/Asynchronous_I/O][3]
[http://stackoverflow.com/questions/748175/asynchronous-vs-synchronous-execution-what-does-it-really-mean][4]
[http://state-threads.sourceforge.net/docs/st.html][5]
[https://en.wikipedia.org/wiki/Coroutine][6]
[https://wiki.openstack.org/wiki/Oslo/blueprints/asyncio][7]
[http://www.stackless.com/][8]
[http://greenlet.readthedocs.org/en/latest/#][9]
[https://blogs.gnome.org/markmc/2013/06/04/async-io-and-python/][10]
[http://www.kegel.com/c10k.html][11]
[https://glyph.twistedmatrix.com/2014/02/unyielding.html][12]
[http://specs.openstack.org/openstack/openstack-specs/specs/eventlet-best-practices.html][13]


 [1]: http://docs.oracle.com/cd/E19455-01/806-5257/6je9h032b/index.html
 [2]: http://concur.rspace.googlecode.com/hg/talk/concur.html#landing-slide
 [3]: https://en.wikipedia.org/wiki/Asynchronous_I/O
 [4]: http://stackoverflow.com/questions/748175/asynchronous-vs-synchronous-execution-what-does-it-really-mean
 [5]: http://state-threads.sourceforge.net/docs/st.html
 [6]: https://en.wikipedia.org/wiki/Coroutine
 [7]: https://wiki.openstack.org/wiki/Oslo/blueprints/asyncio
 [8]: http://www.stackless.com/
 [9]: http://greenlet.readthedocs.org/en/latest/#
 [10]: https://blogs.gnome.org/markmc/2013/06/04/async-io-and-python/
 [11]: http://www.kegel.com/c10k.html
 [12]: https://glyph.twistedmatrix.com/2014/02/unyielding.html
 [13]: http://specs.openstack.org/openstack/openstack-specs/specs/eventlet-best-practices.html
