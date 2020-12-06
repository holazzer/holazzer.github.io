
结论：
1. Manager内没有同步，需要自己管理
2. 我猜测，作者的设计意图是“谁的资源谁管理”：Proxy不是我的资源所以我不管，Proxy创建过程是我负责的，所以只有这里作者加锁了
3. 保持灵活性，同步需求可能比一把锁复杂，所以交给编程者实现

下面是源码分析

```python
>>> from multiprocessing import Manager
>>> Manager
<bound method BaseContext.Manager of <multiprocessing.context.DefaultContext object at 0x0000016D81E789B0>>
```

找到 `multiprocessing.context` 里面

```python
class BaseContext(object):
    def Manager(self):
        '''Returns a manager associated with a running server process

        The managers methods such as `Lock()`, `Condition()` and `Queue()`
        can be used to create shared objects.
        '''
        from .managers import SyncManager
        m = SyncManager(ctx=self.get_context())
        m.start()
        return m
```

所以这里知道了：**引入 multiprocessing 首先自动创建 ctx，Manager是一个 `SyncManager`**

文档也说了，"Returns a started SyncManager object which can be used for sharing objects between processes. The returned manager object corresponds to a spawned child process and has methods which will create shared objects and return corresponding proxies."


从 `multiprocessing.managers`找

```python

class SyncManager(BaseManager):
    '''
    Subclass of `BaseManager` which supports a number of shared object types.

    The types registered are those intended for the synchronization
    of threads, plus `dict`, `list` and `Namespace`.

    The `multiprocessing.Manager()` function creates started instances of
    this class.
    '''

SyncManager.register('Queue', queue.Queue)
SyncManager.register('JoinableQueue', queue.Queue)
SyncManager.register('Event', threading.Event, EventProxy)
SyncManager.register('Lock', threading.Lock, AcquirerProxy)
SyncManager.register('RLock', threading.RLock, AcquirerProxy)
SyncManager.register('Semaphore', threading.Semaphore, AcquirerProxy)
SyncManager.register('BoundedSemaphore', threading.BoundedSemaphore,
                     AcquirerProxy)
SyncManager.register('Condition', threading.Condition, ConditionProxy)
SyncManager.register('Barrier', threading.Barrier, BarrierProxy)
SyncManager.register('Pool', pool.Pool, PoolProxy)
SyncManager.register('list', list, ListProxy)
SyncManager.register('dict', dict, DictProxy)
SyncManager.register('Value', Value, ValueProxy)
SyncManager.register('Array', Array, ArrayProxy)
SyncManager.register('Namespace', Namespace, NamespaceProxy)

# types returned by methods of PoolProxy
SyncManager.register('Iterator', proxytype=IteratorProxy, create_method=False)
SyncManager.register('AsyncResult', create_method=False)
```

可以看到这个 `SyncManager` 什么也没做，所以工作都是在 `BaseManager` 里做的

官方文档也有举例子，Custom Manager：

https://docs.python.org/3.7/library/multiprocessing.html#customized-managers

```python
from multiprocessing.managers import BaseManager

class MathsClass:
    def add(self, x, y):
        return x + y
    def mul(self, x, y):
        return x * y

class MyManager(BaseManager):
    pass

MyManager.register('Maths', MathsClass)

if __name__ == '__main__':
    with MyManager() as manager:
        maths = manager.Maths()
        print(maths.add(4, 3))         # prints 7
        print(maths.mul(7, 8))         # prints 56
```

接下来看 `BaseManager` （我删掉了很多关系不大内容，不影响阅读）

```python
class BaseManager(object):
    '''
    Base class for managers
    '''
    _registry = {}

    def _create(self, typeid, *args, **kwds):
        '''
        Create a new shared object; return the token and exposed tuple
        '''
        assert self._state.value == State.STARTED, 'server not yet started'
        conn = self._Client(self._address, authkey=self._authkey)
        try:
            id, exposed = dispatch(conn, None, 'create', (typeid,)+args, kwds)
        finally:
            conn.close()
        return Token(typeid, self._address, id), exposed

    @classmethod
    def register(cls, typeid, callable=None, proxytype=None, exposed=None,
                 method_to_typeid=None, create_method=True):
        '''
        Register a typeid with the manager type
        '''

        cls._registry[typeid] = (
            callable, exposed, method_to_typeid, proxytype
            )

        if create_method:
            def temp(self, *args, **kwds):
                token, exp = self._create(typeid, *args, **kwds)
                proxy = proxytype(
                    token, self._serializer, manager=self,
                    authkey=self._authkey, exposed=exp
                    )
                conn = self._Client(token.address, authkey=self._authkey)
                dispatch(conn, None, 'decref', (token.id,))
                return proxy
            temp.__name__ = typeid
            setattr(cls, typeid, temp)
```

大概看过之后，发现是 register 函数的 create_method ，给类添加了一个方法 temp，temp 函数返回的是一个 proxy，即本地对象，proxy 会去和 Server 端对象同步



现在去看 proxy
```python
class BaseProxy(object):
    '''
    A base for proxies of shared objects
    '''
    _address_to_local = {}
    _mutex = util.ForkAwareThreadLock()

    def __init__(self, token, serializer, manager=None,
                 authkey=None, expoed=None, incref=True, manager_owned=False):
        with BaseProxy._mutex:  # <-- 唯一一次用到锁，是实例化Proxy的时候
            tls_idset = BaseProxy._address_to_local.get(token.address, None)
            if tls_idset is None:
                tls_idset = util.ForkAwareLocal(), ProcessLocalSet()
                BaseProxy._address_to_local[token.address] = tls_idset
        self._tls = tls_idset[0]
        self._idset = tls_idset[1]
        self._token = token
        self._id = self._token.id
        self._Client = listener_client[serializer][1]

    def _callmethod(self, methodname, args=(), kwds={}):
        '''
        Try to call a method of the referrent and return a copy of the result
        '''
        try:
            conn = self._tls.connection
        except AttributeError:
            util.debug('thread %r does not own a connection',
                       threading.current_thread().name)
            self._connect()
            conn = self._tls.connection

        conn.send((self._id, methodname, args, kwds))
        kind, result = conn.recv()

        if kind == '#RETURN':
            return result
        elif kind == '#PROXY':
            exposed, token = result
            proxytype = self._manager._registry[token.typeid][-1]
            token.address = self._token.address
            proxy = proxytype(
                token, self._serializer, manager=self._manager,
                authkey=self._authkey, exposed=exposed
                )
            conn = self._Client(token.address, authkey=self._authkey)
            dispatch(conn, None, 'decref', (token.id,))
            return proxy
        raise convert_to_error(kind, result)

    def _getvalue(self):
        '''
        Get a copy of the value of the referent
        '''
        return self._callmethod('#GETVALUE')
```

工作流程大致是，对Proxy的操作全部变成按名字调用`_callmethod`，然后有一些约定的操作是以`#`开头的，这个技巧我在 Fluent Python 那本里面看过，发送给Server，Server发回结果，直接返回，或者是更新本地Proxy的状态，这里面没有考虑资源竞争的问题。

我觉得设计者应该是想保持功能简洁，想让使用者自己去加锁，或者其他的机制。

确实官方文档里面没说资源竞争问题，这个我也不知道为什么，可能是作者觉得共享资源默认是要自己加锁来管理，这里我只能猜一下。

作者自己用锁就是在创建Proxy对象的时候，它要求同时不能创建两个Proxy，为了防止一些地址上的混乱问题，所以他给自己安排一把锁，可能这里作者抱着的思想是“谁的资源谁管理”这种。而且，那个 Manager 里面还注册了一堆 threading 里面的同步控制工具，锁、信号量、条件变量，可能设计意图是用Manager创建一个进程间共享的锁/信号量/条件变量等，来对进程间的竞争资源做管理，比如你是这里是很多Replicate进程，每个进程又开很多线程，这些线程基本都一样，所以就得进程之间共享线程锁，问题就变得很复杂，这样设计可以保持灵活性。


