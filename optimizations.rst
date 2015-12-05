*************
Optimizations
*************

Inline function calls
=====================

Example::

    def _get_sep(path):
        if isinstance(path, bytes):
            return b'/'
        else:
            return '/'

    def isabs(s):
        """Test whether a path is absolute"""
        sep = _get_sep(s)
        return s.startswith(sep)

Inline ``_get_sep()`` into ``isabs()`` and simplify the code for the ``str``
type::

    def isabs(s: str):
        return s.startswith('/')

It can be implemented as a simple call to the C function
``PyUnicode_Tailmatch()``.

Note: Inlining uses more memory and disk because the original function should
be kept. Except if the inlined function is unreachable (ex: "private
function"?).

Links:

* `Issue #10399 <http://bugs.python.org/issue10399>`_:
  AST Optimization: inlining of function calls


Move invariants out of the loop
===============================

Example::

    def func(obj, lines):
        for text in lines:
            print(obj.cleanup(text))

Become::

    def func(obj, lines):
        local_print = print
        obj_cleanup = obj.cleanup
        for text in lines:
            local_print(obj_cleanup(text))

Local variables are faster than global variables and the attribute lookup is
only done once.


C functions using only C types
==============================

Optimizations:

* Avoid reference counting
* Memory allocations on the heap
* Release the GIL

Example::

    def demo():
        s = 0
        for i in range(10):
            s += i
        return s

In specialized code, it may be possible to use basic C types like ``char`` or
``int`` instead of Python codes which can be allocated on the stack, instead of
allocating objects on the heap. ``i`` and ``s`` variables are integers in the
range ``[0; 45]`` and so a simple C type ``int`` (or even ``char``) can be
used::

    PyObject *demo(void)
    {
        int s, i;
        Py_BEGIN_ALLOW_THREADS
        s = 0;
        for(i=0; i<10; i++)
            s += i;
        Py_END_ALLOW_THREADS
        return PyLong_FromLong(s);
    }

Note: if the function is slow, we may need to check sometimes if a signal was
received.


Release the GIL
===============

Many methods of builtin types don't need the :ref:`GIL <gil>`. Example:
``"abc".startswith("def")``.


Replace calls to pure functions with the result
===============================================

Examples:

- ``len('abc')`` becomes ``3``
- ``"python2.7".startswith("python")`` becomes ``True``
- ``math.log(32) / math.log(2)`` becomes ``5.0``

Can be implemented in the AST optimizer.


.. _const-prop:

Constant propagation
====================

Propage constant values of variables. Example:

+----------------+----------------------+
| Original       | Constant propagation |
+================+======================+
| ::             | ::                   |
|                |                      |
|   def func()   |   def func()         |
|       x = 1    |       x = 1          |
|       y = x    |       y = 1          |
|       return y |       return 1       |
+----------------+----------------------+

FAT Python implements :ref:`constant propagation <fat-const-prop>`.

Read also the `Wikipedia article on copy propagation
<https://en.wikipedia.org/wiki/Copy_propagation>`_.


.. _const-fold:

Constant folding
================

Compute simple operations at the compilation. Usually, at least arithmetic
operations (a+b, a-b, a*b, etc.) are computed. Example:

+--------------------+------------------+
| Original           | Constant folding |
+====================+==================+
| ::                 | ::               |
|                    |                  |
|   def func()       |   def func()     |
|       return 1 + 1 |       return 2   |
+--------------------+------------------+

FAT Python implements :ref:`constant folding <fat-const-fold>`.

See also

* `issue #1346238 <http://bugs.python.org/issue1346238>`_:
  A constant folding optimization pass for the AST
* `Wikipedia article on constant folding
  <https://en.wikipedia.org/wiki/Constant_folding>`_.


Peephole optimizer
==================

See :ref:`CPython peephole optimizer <cpython-peephole>`.


.. _loop-unroll:

Loop unrolling
==============

Example::

    for i in range(4):
        print(i)

The loop body can be duplicated (twice in this example) to reduce the cost of a
loop::

    for i in range(0,4,2):
        print(i)
        print(i+1)
    i = 3

Or the loop can be removed by duplicating the body for all loop iterations::

    i=0
    print(i)
    i=1
    print(i)
    i=2
    print(i)
    i=3
    print(i)

Combined with other optimizations, the code can be simplified to::

    print('0')
    print('1')
    print('2')
    print('3')
    i = 3

FAT Python implements :ref:`loop unrolling <fat-loop-unroll>`.

Read also the `Wikipedia article on loop unrolling
<https://en.wikipedia.org/wiki/Loop_unrolling>`_.


Remove dead code
================

- ``if DEBUG: print("debug")`` where ``DEBUG`` is known to be False


.. _load-global-optim:

Load globals and builtins when the module is loaded
===================================================

Load globals when the module is loaded? Ex: load "print" name when the module
is loaded.

Example::

    def hello():
        print("Hello World")

Become::

    local_print = print

    def hello():
        local_print("Hello World")

Useful if ``hello()`` is compiled to C code.

FAT Python implements a :ref:`copy builtins to constants optimization
<fat-copy-builtin-to-constant>`.


Don't create Python frames
==========================

Inlining and other optimizations don't create Python frames anymore. It can be
a serious issue to debug programs: tracebacks are an important feature of
Python.

At least in debug mode, frames should be created.

PyPy supports lazy creation of frames if an exception is raised.



