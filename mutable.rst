.. _mutable:

*******************************
Everything in Python is mutable
*******************************

Problem
=======

Developers like Python because it's possible to modify (almost) everything.
This feature is heavily used in unit tests with unittest.mock which can
override builtin function, override class methods, modify "constants, etc.

Most optimization rely on assumptions. For example, inlining rely on the fact
that the inlined function is not modified. Implement optimization in respect of
the Python semantic require to implement various assumptions.

Builtin functions
-----------------

Python provides a lot of builtins functions. All Python applications rely on
them, and usually don't expect that these functions are overriden. In practice,
it is very easy to override them.

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

Technically, the ``len()`` function is loaded in ``func()`` with the
``LOAD_GLOBAL`` instruction which first tries to lookup in frame globals
namespace, and then lookup in the frame builtins namespace.

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

* the ``builtins`` module
* frames have a ``f_builtins`` attribute (builtins dictionary)
* the global ``PyInterpreterState`` structure has a ``builtins`` attribute
  (builtins dictionary)
* frame globals have a ``__builtins__`` variable (builtins dictionary,
  or builtins module when ``__name__`` equals ``__main__``)


Function code
-------------

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

Technically, it is possible to modify local variable of a function outside
the function.

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

A Python module A can be modified by a Python module B.


Multithreading
--------------

When two Python threads are running, the thread B can modify shared resources
of thread A, or even resources which are supposed to only be access by the
thread A like local variables.

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
can make some compromises on the Python semantic to implement more aggressive
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

