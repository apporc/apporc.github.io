nova: 对象的远程使用
==========

### 概述

本文说一下 nova 数据库对象的远程使用。这是 nova 里很精彩的一个地方。
远程对象所实现的效果是，一个 A 服务中的远程对象实例，可以由消息队列传送到
B 服务，B 服务能够使用这个实例，当调用实例的方法时，实际执行这个方法却是在
A 服务中。

以 Instance 类的实例为例，该类是代表虚拟机数据库抽象的类，通常我们通过更改实例
的属性，并调用 Instance.save() 方法更改数据库。该类的定义在
nova/objects/instance.py 中。

nova-conductor 在收到创建虚拟机请求时生成了实例 instance = Instance()，之后将
instance 通过消息队列发送到了 nova-compute，在 nova-compute 进行虚拟机创建的
过程中，经常性地需要更改虚拟机的状态，所以经常出现类似这样的语句：

    instance.task_state = task_states.XXX
    instance.save()

nova-compute 不直接访问数据库，这个 save() 方法，就是由 nova-conductor 来执行的。

### 工作机制

nova-compute 中 instance 的方法调用都不是在本地执行的，实际上这些方法的调用都
转化为对远程 nova-conductor 服务中 instance 对象的 rpc 调用。

这里以 Instance 为例，许多 nova/objects/ 下的类都是同样的工作机制。通过追踪其
save() 方法，我们可以看到它有一个修饰函数：

    @base.remotable
    def save(self, context, expected_vm_state=None,
             expected_task_state=None, admin_state_reset=False):
        ...    ...
这个 base.remotable 就是搞怪的地方。我们追踪到 nova/objects/base.py 中的
remotable() 这个方法。 可以看出，它的骨架是这样的：

    def remotable(fn):
        @functools.wraps(fn)
        def wrapper(self, *args, **kwargs):

            if NovaObject.indirection_api:
                updates, result = NovaObject.indirection_api.object_action(
                    ctxt, self, fn.__name__, args, kwargs)
                return result
            else:
                return fn(self, ctxt, *args, **kwargs)
通过这个修饰函数的作用，当 save() 方法被调用的时候，其行为分入两个叉路，
其中一个是使用 NovaObject.indirection\_api.object\_action() 方法，另一个是
使用原 save() 方法。 而分叉的依据是 NovaObject.indirection\_api 的值。

联想一下，nova-conductor 与 nova-compute 都调用了同样 Instance 类的同样的
save() 方法，其结果却不同，问题显然出在这个分叉路口。通过追踪
indirection\_api 的变化，在 nova/cmd/compute.py 中找到了如下代码：

    if not CONF.conductor.use_local:
        block_db_access()
        objects_base.NovaObject.indirection_api = \
            conductor_rpcapi.ConductorAPI()
nova/cmd/compute.py 是 nova-compute 服务的入口文件，原来这里设置了这个值。
对于 nova-conductor 服务，没有人设置这个值，所以它是默认值 None。

这里有必要了解一下 CONF.conductor.use\_local 这个配置参数，它是用来设置
nova-compute 是否直接访问数据库的。 nova-compute 默认是不能直接访问数据库
的，这个值在 nova/conductor/api.py 中设定，默认值为 False。

    cfg.BoolOpt('use_local',
                default=False,
                help='Perform nova-conductor operations locally'),
如果想让 nova-compute 直接访问数据库，只需要打开这个配置项即可。

#### ConductorAPI

我们沿着 indirection\_api 往下跟踪，看一下远程调用的具体实现。

由 nova/cmd/compute.py 中的设定，我们知道 indirection\_api 是
conductor\_rpcapi.ConductorAPI()，且无论是哪个对象， Instance() 也好，
SecurityGroup() 也好，调用的方法都是 object\_action。代码如下：

    class ConductorAPI(object):
        def __init__(self):
            super(ConductorAPI, self).__init__()
            target = messaging.Target(topic=CONF.conductor.topic, version='2.0')
            ... ...

        def object_action(self, context, objinst, objmethod, args, kwargs):
            cctxt = self.client.prepare()
            return cctxt.call(context, 'object_action', objinst=objinst,
                            objmethod=objmethod, args=args, kwargs=kwargs)

也就是说，这里直接将调用发送到消息队列之中了。需要注意的是几个参数，objinst
是远程对象的实例， objmethod 是想要调用的方法，args 和 kwargs 就不用说了。

这个调用是发往 CONF.conductor.topic 的，默认就是 "conductor"。在这个 topic 上
注册 rpc 方法的是 nova/conductor/manager.py 中的 ConductorManager()，来看它的
object_action 方法。

    def object_action(self, context, objinst, objmethod, args, kwargs):
        """Perform an action on an object."""
        oldobj = objinst.obj_clone()
        result = self._object_dispatch(objinst, objmethod, context,
                                       args, kwargs)

        updates = dict()
        for name, field in objinst.fields.items():
            # 这里判断远程对象的哪些属性更改了，更改了的属性
            # 一起加到 updates 列表之中。
            ... ...
        # 返回更改以及方法的结果。
        return updates, result

    def _object_dispatch(self, target, method, context, args, kwargs):
        try:
            return getattr(target, method)(context, *args, **kwargs)
        except Exception:
            raise messaging.ExpectedException()


方法的参数与 rpcapi 中调用时是一致的，所以很好理解，收到 objinst 和 objmethod
之后，在 nova-conductor 这边调用了 objinst.objmethod()。就是说远程对象是从
rpc 中发过来的，在发过来之前 nova-conductor 这边并没有对应的对象。相当于是说
nova-compute 告诉 nova-conductor，我这里有一个 instance 对象，我把它发给你，
它的那个 save() 方法，你帮我调用一下。

调用完成之后呢，需要将结果返回给 nova-compute，从上面代码可以看到，object\_action()
方法返回了对象的更改以及方法的结果。 来看调用方 nova-compute 如何处理返回。
较完整的 base.remotable 方法。

    def remotable(fn):
        @functools.wraps(fn)
        def wrapper(self, *args, **kwargs):

            if NovaObject.indirection_api:
                updates, result = NovaObject.indirection_api.object_action(
                    ctxt, self, fn.__name__, args, kwargs)
                逐条
                for key, value in updates.iteritems():
                    # 这里将 updates 中的更改，逐条检查更新到 nova-compute
                    # 端的远程对象中。
                    ... ...
                return result

所以概括来说，在调用的过程中，nova-conductor 与 nova-compute 两端共同维护了
远程对象的一致性，确保对象在两边是一模一样的。相当于 nova-conductor 与
nova-compute 两端有一个对象的两个一样的副本，当然对象的一致性是相对的，并不是
每时每刻都一致，更新是需要过程的。

经过以上的梳理，还有一个问题没有提及，那就是对象是如何在 rpc 中传递的。

#### 序列化与反序列化

如果想要对象在 rpc 中传递，显然不能直接拿 python 的内存对象发来发去，消息在
传递的过程中有其独立的格式，例如 xml, json 等。将对象与消息格式之间的转化即
是序列化与反序列化。

nova/objects 中所提供的对象，如果需要支持远程使用的，都必须要提供序列化与反序列化
的方法。它们分别由 obj_from_primitive() 和 obj_to_primitive() 提供。

    class NovaObject(object):
        def obj_to_primitive(self, target_version=None):
            # 序列化方法，将对象转化为原语 ，原语结构在这里本质上是字典。
            primitive = dict()
            for name, field in self.fields.items():
                # 对于对象的每个 field，都分别转化为原语，to_primitive 方法后面再讲。
                if self.obj_attr_is_set(name):
                    primitive[name] = field.to_primitive(self, name,
                                                        getattr(self, name))
            if target_version:
                # 处理版本兼容问题。
                # 呃，远程对象是有版本的，便于软件升级过程中如果数据库有调整，
                # 不同的服务之间仍能协同工作。

                self.obj_make_compatible(primitive, target_version)
            # 最终转化完的结果是下面这个字典结构，可以看到
            # 这里有对象名称，命令空间，版本以及数据内容。
            # 涵盖了对象实例的所有信息，回头根据这些信息就可以重新制作一个一样的对象。
            obj = {'nova_object.name': self.obj_name(),
                'nova_object.namespace': 'nova',
                'nova_object.version': target_version or self.VERSION,
                'nova_object.data': primitive}
            # 远程对象在 rpc 调用过程中，需要记录两侧的更改，便于保持一致性。
            if self.obj_what_changed():
                obj['nova_object.changes'] = list(self.obj_what_changed())
            return obj

        @classmethod
        def obj_from_primitive(cls, primitive, context=None):
            # 反序列化方法，由原语生成对象。这里的工作就是从原语中
            # 获取到对象的信息，然后制作一个原语中描述的对象。
            if primitive['nova_object.namespace'] != 'nova':
                # 支持命令空间检查，对于 nova 来说命令空间必须是 nova。
                raise exception.UnsupportedObjectError(
                    objtype='%s.%s' % (primitive['nova_object.namespace'],
                                    primitive['nova_object.name']))
            # 设定对象的名称
            objname = primitive['nova_object.name']
            # 设定对象的版本
            objver = primitive['nova_object.version']
            # 根据名称与版本，获取到对象的类名。
            objclass = cls.obj_class_from_name(objname, objver)
            # 有了对象的类名，由数据内容生成对象的实例。
            return objclass._obj_from_primitive(context, objver, primitive)

        @classmethod
        def _obj_from_primitive(cls, context, objver, primitive):
            # 解析原语中数据内容，生成实例的方法。
            self = cls()
            self._context = context
            self.VERSION = objver
            objdata = primitive['nova_object.data']
            changes = primitive.get('nova_object.changes', [])
            for name, field in self.fields.items():
                if name in objdata:
                    # 对于原语中的每一项数据，都使用 from_primitive 方法
                    # 转化为一个 field。
                    setattr(self, name, field.from_primitive(self, name,
                                                            objdata[name]))
            self._changed_fields = set([x for x in changes if x in self.fields])
            return self

下面看一下 nova 中的 Field 类。

    class Field(object):

        def __init__(self, field_type, nullable=False,
                    default=UnspecifiedDefault, read_only=False):
            # file_type 是 Integer, String, DataTime 等等的细分类型。
            # 每一种类型都有自己与原语转化的特定方式。
            self._type = field_type

        def from_primitive(self, obj, attr, value):
            if value is None:
                return None
            else:
                # 调用细分类型的转化方法
                return self._type.from_primitive(obj, attr, value)

        def to_primitive(self, obj, attr, value):
            if value is None:
                return None
            else:
                # 调用细分类型的转化方法
                return self._type.to_primitive(obj, attr, value)

对于普通类型，如 Integer，不需要特别的转化。而对于高级类型则需要一些程序，
以 IPAddress 为例：

    class IPAddress(FieldType):
        @staticmethod
        def coerce(obj, attr, value):
            try:
                # 根据字符串初始化一个 IPAddress 类型。
                return netaddr.IPAddress(value)
            except netaddr.AddrFormatError as e:
                raise ValueError(six.text_type(e))

        def from_primitive(self, obj, attr, value):
            return self.coerce(obj, attr, value)

        @staticmethod
        def to_primitive(obj, attr, value):
            # 转化原语时，只需要转化为字符串即可。
            # IPAddress 类型需要提供 __str__ 方法。
            return str(value)
