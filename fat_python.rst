.. _fatpython:

**********
FAT Python
**********

Intro
=====

The FAT Python project was started by Victor Stinner in October 2015 to try to
solve issues of previous attempts of "static optimizers" for Python. The main
feature are efficient guards using versionned dictionaries to check if
something was modified. Guards are used to decide if the specialized bytecode
of a function can be used or not.

Python FAT is expected to be FAT... maybe FAST if we are lucky. FAT because
it will use two versions of some functions where one version is specialised to
specific argument types, a specific environment, optimized when builtins are
not mocked, etc.

Thread on python-ideas: `Add specialized bytecode with guards to functions
<https://mail.python.org/pipermail/python-ideas/2015-October/036908.html>` (Oct
2015).


More information: `FATPYTHON.rst
<https://hg.python.org/sandbox/fatpython/file/tip/FATPYTHON.rst>`_.

Test FAT Python
===============

Download FAT Python with:

    hg clone http://hg.python.org/sandbox/fatpython

Compile it and run tests::

    ./configure && make && ./python -m test test_astoptimizer test_fat

Benchmark::

    ./python bench.py
    ./python -F bench.py

Example
=======

::

    >>> def func():
    ...     return len("abc")
    ...
    >>> import dis
    >>> dis.dis(func)
      2           0 LOAD_GLOBAL              0 (len)
                  3 LOAD_CONST               1 ('abc')
                  6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
                  9 RETURN_VALUE

    >>> len(func.get_specialized())
    1
    >>> specialized=func.get_specialized()[0]
    >>> dis.dis(specialized['code'])
      2           0 LOAD_CONST               1 (3)
                  3 RETURN_VALUE
    >>> len(specialized['guards'])
    2

    >>> func()
    3

    >>> len=lambda obj: "mock"
    >>> func()
    'mock'
    >>> func.get_specialized()
    []

The function ``func()`` has specialized bytecode which returns directly 3
instead of calling ``len("abc")``. The specialized bytecode has two guards
dictionary keys: ``builtins.__dict__['len']`` and ``globals()['len']``. If one
of these keys is modified, the specialized bytecode is simply removed (when the
function is called) and the original bytecode is executed.


Optimizations
=============

Call pure builtins
------------------

Call pure builtin functions at compilation: replace the call with the result in
the specialized bytecode, add guards on the called builtin functions.

The optimization is disabled when the builtin function is modified or if
a variable with the same name is added to the global namespace of the function.

Example::

    >>> def func():
    ...     return len("abc")
    ...
    >>> import dis
    >>> dis.dis(func.get_specialized()[0]['code'])
      2           0 LOAD_CONST               1 (3)
                  3 RETURN_VALUE


Copy builtin functions to constants
-----------------------------------

Opt-in optimization to copy builtin functions to constants.

This optimization is disabled by default because it changes the Python
semantic. Currently, astoptimizer is unable to guess if an instruction can
modify builtins functions or not. For example, the optimization changes the
behaviour of the following function::

    def func():
        x = range(3)
        print(len(x))   # expect: 3

        _len = builtins.len
        try:
            builtins.len = lambda x: "mock"
            print(len(x))   # expect: mock
        finally:
            builtins.len = _len


Goals
=====

Goals:

* *no* overhead when FAT mode is disabled (default). The FAT mode must remain
  optional.
* Faster than current CPython on real applications like Django or Mercurial.
  5% faster would be nice, 10% would be better.
* 100% compatible with CPython and the Python language: everything must be kept
  mutable. Optimizations are disabled when the environment is modified.
* 100% compatible with the CPython C API: ABI and C structures must not be
  modified.
* Add a generic API to support "specialized" functions.

Non-goal:

* FAT Python doesn't modify the Python C API: don't expect better memory
  footprint with specialized types, like PyPy list of integers stored
  as a real array of C int in memory.
* FAT Python is not a JIT. Don't expected crazy performances as PyPy, Numba or
  Pyston. PyPy must remain the fastest implementation of Python, 100%
  compatible with CPython!


Roadmap
=======

Milestone 1: DONE
-----------------

* guard and specialized PoC in Python: DONE
* optimize guards with modification in CPython internals:

  - add version to dictionaries: DONE
  - add version to functions: DONE

* implement guards and specialized function in C: DONE

Expected speedup: 10% on specific microbenchmarks, but require to modify the
source code manually to specialize functions.

Milestone 2: DONE
-----------------

* write an AST optimizer:

  - call pure builtin functions at compilation: DONE
  - generate guards: DONE

* enable the optimizer by default in FAT mode in the site module and
  ensure that *no* test fails in the Python test suite, running
  the test suite with -j0 to isolate processes

Expected speedup: no speedup, it's just a milestone to validate the
implementation. (It's still possible to optimize *manually* code to specialize
functions, to implement better optimizations.)


Milestone 3 (faster)
--------------------

DONE:

* add a configuration to astoptimizer
* opt-in optimization "copy global to locals", currently used to load builtin
  functions

TODO:

* configuration to manually help the optimizer:

  - give a whitelist of "constants": app.DEBUG, app.enum.BLUE, ...
  - type hint with strict types: x is Python int in range [3; 10]
  - expect platform values to be constant: sys.version_info, sys.maxunicode,
    os.name, sys.platform, os.linesep, etc.
  - declare pure functions
  - see astoptimizer for more ideas

* implement more optimizations:

  - constant folding
  - detect pure functions in AST and call them at the compilation
  - function inlining
  - move invariants out of the loop

* implement code to detect the exact type of function parameters and function
  locals and save it into an annotation file
* implement profiling directed optimization: benchmark guards at runtime
  to decide if it's worth to use a specialized function. Measure maybe also
  the memory footprint using tracemalloc?
* implement basic stategy to decide if specialized function must be emitted
  or not using raw estimation, like the size of the bytecode in bytes

Milestone 4 (goal)
------------------

* move the optimizer into a new third-party project. Only keep the API for
  specialized functions with guards


Guards on specialized functions
===============================

To decide if we can use the specialized version of a function, we have to
ensure that the environment was not modified. We will call these checks
"guards.

First attempt: "modified" and "readonly" flags
----------------------------------------------

A first attempt to implement guards was the `readonly PoC
<https://hg.python.org/sandbox/readonly>`_ (fork of CPython) which registered
callbacks to notify all guards. The problem is that modifying a watched
dictionary gets a complexity of O(n) where n is the number of registered
guards.

readonly adds a ``modified`` flag to types and a ``readonly`` property to
dictionaries. The guard was notified with the modified key to decide to disable
or not the optimization.


New try: versionned dictionary
------------------------------

To reduce the cost of dictionary lookup when checking guards, a subclass of
dict is added: verdict(), versionned dictionary. A verdict has a global version
incremented each time that the dict is modified and each mapping (key) has
a version too, modified when a key is modified. Example::

    >>> import fat
    >>> d = fat.verdict()
    >>> d.__version__
    1
    >>> d.getversion('a')
    >>> print(d.getversion('a'))
    None

    >>> d['a'] = 1
    >>> d.__version__
    2
    >>> print(d.getversion('a'))
    2

    >>> d['a'] = 2
    >>> d.__version__
    3
    >>> print(d.getversion('a'))
    3

    >>> del d['a']
    >>> d.__version__
    4
    >>> print(d.getversion('a'))
    None

A guard only has to lookup for the watched key if the global version is
modified. Currently, the specialized function is disabled when the value was
modified, even if the key is modified and restored before the guard is checked.
The reason for this is that keeping a reference to the watched value can create
reference leaks and may keep objects alive longer than expected.

For the same reason, the guard doesn't keep a strong reference to the
dictionary, but a *weak* reference. It's not possible to create a weak
reference to a dict, but it's possible to create a weak reference to a verdict.

Guards
------

Guards:

* FuncGuard: check if a function was modified (currently only __code__ is
  checked)
* DictGuard: check if a dictionary key is created (if it didn't exist) or
  modified
* ArgTypeGuard: check the type of function arguments

Example: Guard on a builtin function
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Example of function::

    def use_builtin_len():
        return len("abc")

To replace ``len("abc")``, we have to ensure that:

* the builtin ``len()`` function was not overriden
  with ``builtins.len = mock_len``
* the ``len`` symbol was not added to the function globals which are the module
  globals

Example: Guard to inline a function
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Example of function::

    def is_python(filename):
        return filename.endswith('.py')

    def filter_python(filenames):
        return [filename for filename in filenames
                if is_python(filename)]

To replace ``is_python(filename)`` with ``filename.endswith('.py')`` in
``filter_python()``, we have to ensure that:

* the ``is_python`` symbol was not modified in the namespace (module globals)
* the ``is_python()`` function was not modified


Specialized methods
===================

FIXME: this section must be written, it looks wrong.

A specialized method requires to be more careful, guards must be put on the
object method but also on the class method.

See the fat.specialized_method() function.


Implementation
==============

FAT python:

* Object/dictobject.c: implementation of the new verdict() type
* Modules/_fat.c: specialized functions with guards
* Lib/fat.py: guards and specialized function (_fat part not implemented
  in C yet)
* Tests

  - Lib/test/fattester.py
  - Lib/test/test_fat.py

Other changes:

* Objects/moduleobject.c: use verdict for module dictionaries
* Python/bltinmodule.c: add __fat__ builtin symbol
* Python/ceval.c: bugfix when builtins is not a dict type
* Python/sysmodule.c: add sys.flags.fat
* Modules/main.c: add -F command line option

See also ASTOPTIMIZER.rst for the documentation on the AST optimizer.


Possible optimizations
======================

Short term:

* Function func2() calls func1() if func1() is pure: inline func1()
  into func2()
* Call builtin pure functions during compilation. Example: replace len("abc")
  with 3 or range(3) with (0, 1, 2).
* Constant folding: replace a variable with its value. We may do that for
  optimal parameters with default value if these parameters are not set.
  Example: replace app.DEBUG with False.

Using types:

* Detect the exact type of parameters and function local variables
* Specialized code relying on the types. For example, move invariant out of
  loops (ex: obj.append for list).
* x + 0 gives a TypeError for str, but can be replaced with x for int and
  float. Same optimization for x*0.
* See astoptimizer for more ideas.

Longer term:

* Compile to machine code using Cython, Numba, PyPy, etc. Maybe only for
  numeric types at the beginning? Release the GIL if possible, but check
  "sometimes" if we got UNIX signals.


Pure functions
==============

A "pure" function is a function with no side effect.

Example of pure operators:

* x+y, x-y, x*y, x/y, x//y, x**y for types int, float, complex, bytes, str,
  and also tuple and list for x+y

Example of instructions with side effect:

* "global var"

Example of pure function::

    def mysum(x, y):
        return x + y

Example of function with side effect::

    global _last_sum

    def mysum(x, y):
        global _last_sum
        s = x + y
        _last_sum = s
        return s


Expected limitations
====================

Inlining makes debugging more complex:

* sys.getframe()
* locals()
* pdb
* etc.
* don't work as expected anymore

Bugs, shit happens:

* Missing guard: specialized function is called even if the "environment"
  was modified

FAT python! Memory vs CPU, fight!

* Memory footprint: loading two versions of a function is memory uses more
  memory
* Disk usage: .pyc will be more larger

Possible worse performance:

* guards adds an overhead higher than the optimization of the specialized code
* specialized code may be slower than the original bytecode


FAT Python API
==============

* func.specialize(function[, guards: list]) -> func_index:
  guards a list of dict. Guards:

  - {'guard_type': 'func', 'func': func2}:
    guard on func2.__code__
  - {'guard_type': 'builtins', 'name': 'len'}:
    guard on builtins.__dict__['len']
  - {'guard_type': 'globals', 'name': 'obj'}:
    guard on globals()['obj']
  - {'guard_type': 'type', 'type': MyClass, 'key': 'attr'}:
    guard on MyClass.__dict__['attr']
  - {'guard_type': 'arg_type', 'arg_index': 0, 'type': str}:
    type of the function argument 0 must be str

* func.specialize(bytecode[, guards: list]) -> func_index
* func.add_arg_type_guard(func_index, arg_index, type)
* func.add_builtin_guard(func_index, symbol): guard on getattr(builtins, key)
* func.add_dict_guard(func_index, dict, key): guard on dict[key]
* func.add_func_guard(func_index, func2): guard on func2.__code__
* func.add_global_guard(func_index, key): guard on globals()[key]
* func.add_type_dict_guard(func_index, type, key): guard on type.__dict__[key]
* func.get_specialized()

For dictionary and function guards: specialized functions are removed if the
guards fail:

* Broken weak-reference to the dictionary/function
* The dictionary key was modified (created, modified or removed depending on
  the initial state)
* The function was modified
* An error occurred when getting the dictionary entry to get the key version


Effect of FAT Python
====================

* Use fat.verdict instead of dict for:

  * module.__dict__
  * my_class.__dict__
  * my_instance.__dict__
  * set __fat__ to True


astoptimizer
============

* Lib/astoptimizer.py
* add sys.asthook
* PyParser_ASTFromStringObject() calls call_ast_hook()
* Different .pyc filename:

  - Lib/__pycache__/os.cpython-36.pyc: default mode
  - Lib/__pycache__/os.cpython-36.fat-0.pyc: FAT mode


Misc notes
==========

Optimizations are not disabled when tracing is enabled.
