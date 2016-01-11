.. _old-ast-optimizer:

+++++++++++++++++
Old AST Optimizer
+++++++++++++++++

See also :ref:`AST optimizers <ast-optimizers>`.

https://bitbucket.org/haypo/astoptimizer/ was a first attempt to optimize
Python. This project was rejected by the Python community because it breaks the
Python semantics. For example, it replaces ``len("abc")`` with ``3``. It checks
that ``len()`` was not overriden in the module, but it doesn't check that the
builtin ``len()`` function was not overriden.

Threads on the Python-Dev mailing list:

* `AST optimizer implemented in Python
  <https://mail.python.org/pipermail/python-dev/2012-August/121286.html>`_.
* `Release of astoptimizer 0.3
  <https://mail.python.org/pipermail/python-dev/2012-September/121647.html>`_:
  Guido van Rossum, Nick Coghlan and Maciej Fijalkowski complained that too
  many optimizations broke the Python semantic

The project was created in September 2012. It is now dead and replaced with the
:ref:`new AST optimizer <new-ast-optimizer>`.


Introduction
============

astoptimizer is an optimizer for Python code working on the Abstract Syntax
Tree (AST, high-level representration). It does as much work as possible at
compile time.

The compiler is static, it is not a just-in-time (JIT) compiler, and so don't
expect better performances than psyco or PyPy for example. Optimizations
depending on the type of functions parameters cannot be done for examples.
Only optimizations on immutable types (constants) are done.

Website: http://pypi.python.org/pypi/astoptimizer

Source code hosted at: https://bitbucket.org/haypo/astoptimizer


Optimizations
=============

* Call builtin functions if arguments are constants (need "builtin_funcs"
  feature). Examples:

  - ``len("abc")`` => ``3``
  - ``ord("A")`` => ``65``

* Call methods of builtin types if the object and arguments are constants.
  Examples:

  - ``u"h\\xe9ho".encode("utf-8")`` => ``b"h\\xc3\\xa9ho"``
  - ``"python2.7".startswith("python")`` => ``True``
  - ``(32).bit_length()`` => ``6``
  - ``float.fromhex("0x1.8p+0")`` => ``1.5``

* Call functions of math and string modules for functions without
  border effect. Examples:

  - ``math.log(32) / math.log(2)`` => ``5.0``
  - ``string.atoi("5")`` => ``5``

* Format strings for str%args and print(arg1, arg2, ...) if arguments
  are constants and the format string is valid.
  Examples:

  - ``"x=%s" % 5`` => ``"x=5"``
  - ``print(1.5)`` => ``print("1.5")``

* Simplify expressions. Examples:

  - ``not(x in y)`` => ``x not in y``
  - ``4 and 5 and x and 6`` => ``x and 6``
  - ``if a: if b: print("true")`` => ``if a and b: print("true")``

* Optimize loops (range => xrange needs "builtin_funcs" features). Examples:

  - ``while True: pass`` => ``while 1: pass``
  - ``for x in range(3): print(x)`` => ``x = 0; print(x); x = 1; print(x); x = 2; print(x)``
  - ``for x in range(1000): print(x)`` => ``for x in xrange(1000): print(x)`` (Python 2)

* Optimize iterators, list, set and dict comprehension, and generators (need
  "builtin_funcs" feature). Examples:

  - ``iter(set())`` => ``iter(())``
  - ``frozenset("")`` => ``frozenset()``
  - ``(x for x in "abc" if False)`` => ``(None for x in ())``
  - ``[x*10 for x in range(1, 4)]`` => ``[10, 20, 30]``
  - ``(x*2 for x in "abc" if True)`` => ``(x*2 for x in ("a", "b", "c"))``
  - ``list(x for x in iterable)`` => ``list(iterable)``
  - ``tuple(x for x in "abc")`` => ``("a", "b", "c")``
  - ``list(x for x in range(3))`` => ``[0, 1, 2]``
  - ``[x for x in ""]`` => ``[]``
  - ``[x for x in iterable]`` => ``list(iterable)``
  - ``set([x for x in "abc"])`` => ``{"a", "b", "c"}`` (Python 2.7+) or ``set(("a", "b", "c"))``

* Replace list with tuple (need "builtin_funcs" feature). Examples:

  - ``for x in [a, b, c]: print(x)`` => ``for x in (a, b, c): print(x)``
  - ``x in [1, 2, 3]`` => ``x in (1, 2, 3)``
  - ``list([x, y, z])`` => ``[x, y, z]``
  - ``set([1, 2, 3])`` => ``{1, 2, 3}`` (Python 2.7+)

* Evaluate unary and binary operators, subscript and comparaison if all
  arguments are constants. Examples:

  - ``1 + 2 * 3`` => ``7``
  - ``not True`` => ``False``
  - ``"abc" * 3`` => ``"abcabcabc"``
  - ``"abcdef"[:3]`` => ``"abc"``
  - ``(2, 7, 3)[1]`` => ``7``
  - ``frozenset("ab") | frozenset("bc")`` => ``frozenset("abc")``
  - ``None is None`` => ``True``
  - ``"2" in "python2.7"`` => ``True``
  - ``x in [1, 2, 3]`` => ``x in {1, 2, 3}`` (Python 3) or ``x in (1, 2, 3)`` (Python 2)
  - ``def f(): return 2 if 4 < 5 else 3`` => ``def f(): return 2``

* Remove empty loop. Example:

  - ``for i in (1, 2, 3): pass`` => ``i = 3``

* Remove dead code. Examples:

  - ``def f(): return 1; return 2`` => ``def f(): return 1``
  - ``def f(a, b): s = a+b; 3; return s`` => ``def f(a, b): s = a+b; return s``
  - ``if DEBUG: print("debug")`` => ``pass`` with DEBUG declared as False
  - ``while 0: print("never executed")`` => ``pass``


Use astoptimizer in your project
================================

To enable astoptimizer globally on your project, add the following lines at the
very begining of your application::

    import astoptimizer
    config = astoptimizer.Config('builtin_funcs', 'pythonbin')
    # customize the config here
    astoptimizer.patch_compile(config)

On Python 3.3, imports will then use the patched compile() function and so
all modules will be optimized. With older versions, the compileall module
(ex: compileall.compile_dir()) can be used to compile an application
with optimizations enabled.

See also the issue `#17515: Add sys.setasthook() to allow to use
a custom AST optimizer <http://bugs.python.org/issue17515>`_.


Example
=======

Example with the high-level function ``optimize_code``::

    from astoptimizer import optimize_code
    code = "print(1+1)"
    code = optimize_code(code)
    exec(code)

Example the low-level functions ``optimize_ast``::

    from astoptimizer import Config, parse_ast, optimize_ast, compile_ast
    config = Config('builtin_funcs', 'pythonbin')
    code = "print(1+1)"
    tree = parse_ast(code)
    tree = optimize_ast(tree, config)
    code = compile_ast(tree)
    exec(code)

See also ``demo.py`` script.


Configuration
=============

Unsafe optimizations are disabled by default. Use the Config() class to enable
more optimizations.

Features enabled by default:

* ``"builtin_types"``: methods of bytes, str, unicode, tuple, frozenset, int
  and float types
* ``"math"``, ``"string"``: constants and functions without border effects of
  the math / string module

Optional features:

* ``"builtin_funcs"``: builtin functions like abs(), str(), len(), etc. Examples:

  - ``len("abc")`` => ``3``
  - ``ord("A")`` => ``65``
  - ``str(123)`` => ``"123"``

* ``"pythonbin"``: Enable this feature if the optimized code will be executed by
  the same Python binary: so exactly the same Python version with the same
  build options. Allow to optimize non-BMP unicode strings on Python < 3.3.
  Enable the ``"platform"`` feature. Examples:

  - ``u"\\U0010ffff"[0]`` => ``u"\\udbff"`` or ``u"\\U0010ffff"`` (depending on
    build options, narrow or wide Unicode)
  - ``sys.version_info.major`` => ``2``
  - ``sys.maxunicode`` => ``0x10ffff``

* ``"pythonenv"``: Enable this feature if you control the environment
  variables (like ``PYTHONOPTIMIZE``) and Python command line options (like
  ``-Qnew``).  On Python 2, allow to optimize int/int. Enable ``"platform"``
  and ``"pythonbin"`` features. Examples:

  - ``__debug__`` => ``True``
  - ``sys.flags.optimize`` => ``0``

* ``"platform"``: optimizations specific to a platform. Examples:

  - ``sys.platform`` => ``"linux2"``
  - ``sys.byteorder`` => ``"little"``
  - ``sys.maxint`` => ``2147483647``
  - ``os.linesep`` => ``"\\n"``

* ``"struct"``: struct module, calcsize(), pack() and unpack() functions.

* ``"cpython_tests"``: disable some optimizations to workaround issues with
  the CPython test suite. Only use it for tests.

Use ``Config("builtin_funcs", "pythonbin")`` to enable most optimizations.  You
may also enable ``"pythonenv"`` to enable more optimizations, but then the
optimized code will depends on environment variables and Python command line
options.

Use config.enable_all_optimizations() to enable all optimizations, which may
generate invalid code.


Advices
=======

Advices to help the AST optimizer:

* Declare your constants using config.add_constant()
* Declare your pure functions (functions with no border effect) using
  config.add_func()
* Don't use "from module import \*". If "import \*" is used, builtins
  functions are not optimized anymore for example.


Limitations
===========

* Operations on mutable values are not optimized, ex: len([1, 2, 3]).
* Unsafe optimizations are disabled by default. For example, len("\\U0010ffff") is not
  optimized because the result depends on the build options of Python. Enable
  "builtin_funcs" and "pythonenv" features to enable more optimizations.
* len() is not optimized if the result is bigger than 2^31-1.
  Enable "pythonbin" configuration feature to optimize the call for bigger
  objects.
* On Python 2, operators taking a bytes string and a unicode string are not
  optimized if the bytes string has to be decoded from the default encoding or
  if the unicode string has to be encoded to the default encoding. Exception:
  pure ASCII strings are optimized. For example, b"abc" + u"def" is replaced
  with u"abcdef", whereas u"x=%s" % b"\\xe9" is not optimized.
* On Python 3, comparaison between bytes and Unicode strings are not optimized
  because the comparaison may emit a warning or raise a BytesWarning
  exception. Bytes string are not converted to Unicode string. For example,
  b"abc" < "abc" and str(b"abc") are not optimized. Converting a bytes string
  to Unicode is never optimized.


ChangeLog
=========

Version 0.6 (2014-03-05)
------------------------

* Remove empty loop. Example:
  ``for i in (1, 2, 3): pass`` => ``i = 3``.
* Log removal of code
* Fix support of Python 3.4: socket constants are now enum

Version 0.5 (2013-03-26)
------------------------

* Unroll loops (no support for break/continue yet) and list comprehension.
  Example: ``[x*10 for x in range(1, 4)]`` => ``[10, 20, 30]``.
* Add Config.enable_all_optimizations() method
* Add a more aggressive option to remove dead code
  (config.remove_almost_dead_code), disabled by default
* Remove useless instructions. Example:
  "x=1; 'abc'; print(x)" => "x=1; print(x)"
* Remove empty try/except. Example:
  "try: pass except: pass" => "pass"

Version 0.4 (2012-12-10)
------------------------

Bugfixes:

* Don't replace range() with xrange() if arguments cannot be converted to C
  long
* Disable float.fromhex() optimization by default: float may be shadowed.
  Use "builtin_funcs" to enable this optimization.

Changes:

* Add the "struct" configuration feature: functions of the struct module
* Optimize print() on Python 2 with "from __future__ import print_function"
* Optimize iterators, list, set and dict comprehension, and generators
* Replace list with tuple
* Optimize ``if a: if b: print("true")``: ``if a and b: print("true")``

Version 0.3.1 (2012-09-12)
--------------------------

Bugfixes:

* Disable optimizations on functions and constants if a variable with the same
  name is set. Example: "len=ord; print(len('A'))",
  "sys.version = 'abc'; print(sys.version)".
* Don't optimize print() function, frozenset() nor range() functions if
  "builtin_funcs" feature is disabled
* Don't remove code if it contains global or nonlocal.
  Example: "def f(): if 0: global x; x = 2".

Version 0.3 (2012-09-11)
------------------------

Major changes:

* Add astoptimizer.patch_compile(config=None) function to simply hook the
  builtin compile() function.
* Add "pythonbin" configuration feature.
* Disable optimizations on builtin functions by default. Add "builtin_funcs"
  feature to the configuration to optimize builtin functions.
* Remove dead code (optionnal optimization)
* It is now posible to define a callback for warnings of the optimizer
* Drop support of Python 2.5, it is unable to compile an AST tree to bytecode.
  AST objects of Python 2.5 don't accept arguments in constructors.

Bugfixes:

* Handle "from math import \*" correctly
* Don't optimize operations if arguments are bytes and unicode strings.
  Only optimize if string arguments have the same type.
* Disable optimizations on non-BMP unicode strings by default. Optimizations
  enabled with "pythonbin" feature.

Other changes:

* More functions, methods and constants:

  - bytes, str, unicode: add more methods.
  - math module: add most remaining functions
  - string module: add some functions and all constants

* not(a in b) => a not in b, not(a is b) => a is not b
* a if bool else b
* for x in range(n) => for x in xrange(n) (only on Python 2)
* Enable more optimizations if a function is not a generator
* Add sys.flags.<attr> and sys.version_info.<attr> constants

Version 0.2 (2012-09-02)
------------------------

Major changes:

* Check input arguments before calling an operator or a function, instead of
  catching errors.
* New helper functions optimize_code() and optimize_ast() should be used
  instead of using directly the Optimizer class.
* Support tuple and frozenset types

Changes:

* FIX: add Config.max_size to check len(obj) result
* FIX: disable non portable optimizations on non-BMP strings
* Support Python 2.5-3.3
* Refactor Optimizer: Optimizer.visit() now always visit children before
  calling the optimizer for a node, except for assignments
* Float and complex numbers are no more restricted by the integer range of the
  configuration
* More builtin functions. Examples: divmod(int, int), float(str), min(tuple),
  sum(tuple).
* More method of builtin types. Examples: str.startswith(), str.find(),
  tuple.count(), float.is_integer().
* math module: add math.ceil(), math.floor() and math.trunc().
* More module constants. Examples: os.O_RDONLY, errno.EINVAL,
  socket.SOCK_STREAM.
* More operators: a not in b, a is b, a is not b, +a.
* Conversion to string: str(), str % args and print(arg1, arg2, ...).
* Support import aliases. Examples: "import math as M; print(M.floor(1.5))"
  and "from math import floor as F; print(F(1.5))".
* Experimental support of variables (disabled by default).

Version 0.1 (2012-08-12)
------------------------

* First public version (to reserve the name on PyPI!)

