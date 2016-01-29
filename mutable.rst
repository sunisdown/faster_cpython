.. _mutable:

*******************************
Everything in Python is mutable
*******************************

Problem
=======

开发者之所以喜欢用 `Python`，很大一部分原因是因为 `Python` 很灵活。
比如在单元测试里面，经常会用 `unittest.mock` ，这样可以覆盖一些内置函数，覆盖类的方法，各种灵活的用法。

Developers like Python because it's possible to modify (almost) everything.
This feature is heavily used in unit tests with unittest.mock which can
override builtin function, override class methods, modify "constants, etc.

Most optimization rely on assumptions. For example, inlining rely on the fact
that the inlined function is not modified. Implement optimization in respect of
the Python semantics require to implement various assumptions.

Builtin functions
-----------------

`Python` 提供的很多内置函数。
每一个用 `Python` 写的代码里面都会用到这些内置函数，但是通常却没有意识到这些函数已经被覆盖掉了。
实际上，这种情况非常容易发生.

Python provides a lot of builtins functions. All Python applications rely on
them, and usually don't expect that these functions are overriden. In practice,
it is very easy to override them.

用 ``len()`` 函数来举个例子::

Example overriden the builtin ``len()`` function::

    import builtins

    def func(obj):
        print("length: %s" % len(obj))

    func("abc")
    builtins.len = lambda obj: "mock!"
    func("abc")

Output::

    length: 3
    length: mock!

从上面的代码里面我们可以看到 ``len()`` 函数被加载到了 ``func`` 里面，按照 ``LEGB` 的原则，
会先在 ``blobals`` 的命名空间中查找 ``len``, 失败后会尝试在 ``builtins`` 里面继续查找。

Technically, the ``len()`` function is loaded in ``func()`` with the
``LOAD_GLOBAL`` instruction which first tries to lookup in frame globals
namespace, and then lookup in the frame builtins namespace.

下面这个例子里面尝试用 ``global`` 命名空间里面的 ``len`` 函数来覆盖调 ``buildin`` 里面的函数:

Example overriding the ``len()`` builtin function with a ``len()`` function
injected in the global namespace::

    def func(obj):
        print("length: %s" % len(obj))

    func("abc")
    len = lambda obj: "mock!"
    func("abc")

Output::

    length: 3
    length: mock!

Builtins are references in multiple places:

``Builtins`` 在很多地方都有被提及:

* the ``builtins`` module
* frames have a ``f_builtins`` attribute (builtins dictionary)
* the global ``PyInterpreterState`` structure has a ``builtins`` attribute
  (builtins dictionary)
* frame globals have a ``__builtins__`` variable (builtins dictionary,
  or builtins module when ``__name__`` equals ``__main__``)


Function code
-------------

还可以把整个函数的行为都作出修改::

It is possible to modify at runtime the bytecode of a function to modify
completly its behaviour. Example::

    def func(x, y):
        return x + y

    print("1+2 = %s" % func(1, 2))

    def mock(x, y):
        return 'mock'

    func.__code__ = mock.__code__
    print("1+2 = %s" % func(1, 2))

Output::

    1+2 = 3
    1+2 = mock

Local variables
---------------

还可以用比较 ``hack`` 的方法来让一个函数来修改这个函数外面的 ``local variable``

Technically, it is possible to modify local variable of a function outside
the function.

举个例子: 我们用 ``hack()`` 来修改调用它的函数的 ``local variable``, 变量名为 ``x``

Example of a function ``hack()`` which modifies the ``x`` local variable of its
caller::

    import sys
    import ctypes

    def hack():
        # Get the frame object of the caller
        frame = sys._getframe(1)
        frame.f_locals['x'] = "hack!"
        # Force an update of locals array from locals dict
        ctypes.pythonapi.PyFrame_LocalsToFast(ctypes.py_object(frame),
                                              ctypes.c_int(0))

    def func():
        x = 1
        hack()
        print(x)

    func()

Output::

    hack!


Modification made from other modules
------------------------------------

一个 ``Python`` 的模块 `A` 可以被模块 `B` 修改

A Python module A can be modified by a Python module B.


Multithreading
--------------

同时运行两个 ``Python`` 线程的时候，线程 ``B`` 可以修改线程 ``A`` 的共享资源。
即使是像 ``local variables`` 这样只有线程 ``A`` 可以访问的资源，仍然可以被 ``B`` 修改

When two Python threads are running, the thread B can modify shared resources
of thread A, or even resources which are supposed to only be access by the
thread A like local variables.

线程 ``B`` 可以修改函数代码，覆盖内置函数，修改 ``local variables`` 等

The thread B can modify function code, override builtin functions, modify local
variables, etc.

Python Imports and Python Modules
---------------------------------

The Python import path ``sys.path`` is initialized by multiple environment
variables (ex: ``PYTHONPATH`` and ``PYTHONHOME``), modified by the ``site``
module and can be modified anytime at runtime (by modifying ``sys.path``
directly).

Moreover, it is possible to modify ``sys.modules`` which is the "cache" between
a module fully qualified name and the module object. For example,
``sys.modules['sys']`` should be ``sys``. It is posible to remove modules
from ``sys.modules`` to force to reload a module. It is possible to replace
a module in ``sys.modules``.

The eventlet modules injects monkey-patched modules in ``sys.modules`` to
convert I/O blocking operations to asynchronous operations using an event loop.


Solutions
=========

Make strong assumptions, ignore changes
---------------------------------------

If the optimizer is an opt-in options, users are aware that the optimizer
can make some compromises on the Python semantics to implement more aggressive
optimizations.


Static analysis
---------------

Analyze the code to ensure that functions don't mutate everything, for example
ensure that a function is pure.

Dummy example::

    def func(x, y):
        return x + y

This function ``func()`` is pure: it has no side effect. This function will not
override builtins, not modify local variables of the caller, etc. It is safe to
call this function from anywhere.

It is possible to analyze the code to check that an optimization can be
enabled.


Use guards checked at runtime
-----------------------------

For some optimizations, a static analysis cannot ensure that all assumptions
required by an optimization will respected. Adding guards allows to check
assumptions during the execution to use the optimized code or fallback to the
original code.
