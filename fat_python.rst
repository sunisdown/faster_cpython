.. _fat-python:

**********
FAT Python
**********

.. image:: fat_python.jpg
   :alt: Three-year-old Cambodian boy Oeun Sambat hugs his best friend, a four-metre long female python named Chamreun
   :align: right
   :target: http://pictures.reuters.com/archive/CAMBODIA-PYTHONBOY-RP3DRIMPKQAA.html

.. Source of the photo:
   Three-year-old befriends python
   Sit Tbow (Cambodia) May 22
   Cambodians are flocking to see a three-year-old boy they believe was the son
   of a dragon in his previous life because his best friend is a
   four-metre-long python.
   Curled up for an afternoon snooze inside the coils of his companion, the
   child, Oeun Sambath, attracts regular visits from villagers anxious to make
   use of what they believe are his supernatural powers. "He has been playing
   with the python ever since he could first crawl," said his mother Kim
   Kannara. Reuters

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

The FAT Python project is made of multiple parts:

* ``fat`` module: `fat project at GitHub <https://github.com/haypo/fat>`_.
  C extension implementing different guards.
* ``fatoptimizer`` module: `fatoptimizer project at GitHub
  <https://github.com/haypo/fatoptimizer>`_. AST optimizer implementing
  multiple optimizations and can specialize functions using guards of the
  ``fat`` module. Optimized code depends on the ``fat`` module (if at
  least one function was specialized).
* Python 3.6 patched with the patches for the PEP 509 (dictionary versioning),
  PEP 510 (specialize functions) and PEP 511 (AST transformers)

The project was created in October 2015.

See also the :ref:`AST optimizer <new-ast-optimizer>`.


Status
======

FAT Python PEPs:

* PEP 509: `Add a private version to dict
  <https://www.python.org/dev/peps/pep-0509/>`_
* PEP 510: `Specialized functions with guards
  <https://www.python.org/dev/peps/pep-0510/>`_
* PEP 511: `API for AST transformers
  <https://www.python.org/dev/peps/pep-0511/>`_

Announcements and status reports:

* `'FAT' and fast: What's next for Python
  <http://www.infoworld.com/article/3020450/application-development/fat-fast-whats-next-for-python.html>`_:
  Article of InfoWorld by Serdar Yegulalp (January 11, 2016)
* [Python-Dev] `Third milestone of FAT Python
  <https://mail.python.org/pipermail/python-dev/2015-December/142397.html>`_
* `Status of the FAT Python project, November 26, 2015
  <https://haypo.github.io/fat-python-status-nov26-2015.html>`_
* [python-dev] `Second milestone of FAT Python
  <https://mail.python.org/pipermail/python-dev/2015-November/142113.html>`_
  (Nov 2015)
* [python-ideas] `Add specialized bytecode with guards to functions
  <https://mail.python.org/pipermail/python-ideas/2015-October/036908.html>`_
  (Oct 2015)


fat module
==========

The ``fat`` module:

* `fat module at GitHub
  <https://github.com/haypo/fat>`_
* `fat module at the Python Cheeseshop (PyPI)
  <https://pypi.python.org/pypi/fat>`_


Test FAT Python
===============

Download FAT Python with::

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


TODO list
=========

See the `FAT Python TODO.rst file
<https://hg.python.org/sandbox/fatpython/file/tip/TODO.rst>`_.


.. _fat-optim:

Optimizations
=============

Implementated optimizations:

* :ref:`Call pure builtins <fat-call-pure>`
* :ref:`Loop unrolling <fat-loop-unroll>`
* :ref:`Constant propagation <fat-const-prop>`
* :ref:`Constant folding <fat-const-fold>`
* :ref:`Replace builtin constants <fat-replace-builtin-constant>`
* :ref:`Dead code elimination <fat-dead-code>`
* :ref:`Copy builtin functions to constants <fat-copy-builtin-to-constant>`
* :ref:`Simplify iterable <fat-simplify-iterable>`


.. _fat-call-pure:

Call pure builtins
------------------

Call pure builtin functions at compilation: replace the call with the result in
the specialized bytecode, add guards on the called builtin functions.

The optimization is disabled when the builtin function is modified or if
a variable with the same name is added to the global namespace of the function.

The optimization on the builtin ``NAME`` requires two guards:

* ``NAME`` key in builtin namespace
* ``NAME`` key in global namespace

Example:

+------------------------+---------------+
| Original               | Specialized   |
+========================+===============+
| ::                     | ::            |
|                        |               |
|  def func():           |  def func():  |
|      return len("abc") |      return 3 |
+------------------------+---------------+


.. _fat-loop-unroll:

Loop unrolling
--------------

``for i in range(3): ...`` and ``for i in (1, 2, 3): ...`` are unrolled.
By default, only loops with 16 iterations or less are optimized.

.. note::
   If ``break`` and/or ``continue`` instructions are used in the loop body,
   the loop is not unrolled.

:ref:`Configuration option <fat-config>`: ``unroll_loops``.

See also the :ref:`loop unrolling optimization <loop-unroll>`.

tuple example
^^^^^^^^^^^^^

Example with a tuple.

+---------------------------+--------------------------+
| Original                  | Loop unrolled            |
+===========================+==========================+
| ::                        | ::                       |
|                           |                          |
|  def func():              |  def func():             |
|      for i in ("a", "b"): |      i = "a"             |
|          print(i)         |      print(i)            |
|                           |                          |
|                           |      i = "b"             |
|                           |      print(i)            |
+---------------------------+--------------------------+

No guard is required. The function has no specialized bytecode, the
optimization is done directly on the function.

Original bytecode::

    .     0 SETUP_LOOP              14 (to 17)
          3 LOAD_CONST               3 (('hello', 'world'))
          6 GET_ITER

    >>    7 FOR_ITER                 6 (to 16)
         10 STORE_FAST               0 (i)

         13 JUMP_ABSOLUTE            7
    >>   16 POP_BLOCK

    >>   17 LOAD_CONST               0 (None)
         20 RETURN_VALUE

FAT Python bytecode::

    LOAD_CONST   1 ("hello")
    STORE_FAST   0 (i)

    LOAD_CONST   2 ("world")
    STORE_FAST   0 (i)

    LOAD_CONST   0 (None)
    RETURN_VALUE


range example
^^^^^^^^^^^^^

Example of a loop using ``range()``.

+--------------------------+--------------------------+
| Original                 | Loop unrolled            |
+==========================+==========================+
| ::                       | ::                       |
|                          |                          |
|  def func():             |  def func():             |
|      for i in range(2):  |      i = 0               |
|          print(i)        |      print(i)            |
|                          |                          |
|                          |      i = 1               |
|                          |      print(i)            |
+--------------------------+--------------------------+

The specialized bytecode requires two :ref:`guards <fat-guard>`:

* ``range`` builtin variable
* ``range`` global variable

Combined with :ref:`constant propagation <fat-const-prop>`, the code becomes
even more interesting::

    def func():
        i = 0
        print(0)

        i = 1
        print(1)


.. _fat-const-prop:

Constant propagation
--------------------

Propagate constant values of variables.

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

:ref:`Configuration option <fat-config>`: ``constant_propagation``.

See also the :ref:`constant propagation <const-prop>` optimization.


.. _fat-const-fold:

Constant folding
----------------

Compute simple operations at the compilation:

* arithmetic operations:

  - ``a+b``, ``a-b``, ``a*b``, ``a/b``: int, float, complex
  - ``+x``, ``-x``, ``~x``: int, float, complex
  - ``a//b``, ``a%b``, ``a**b``: int, float
  - ``a<<b``, ``a>>b``, ``a&b``, ``a|b``, ``a^b``: int

* comparison, tests:

  - ``a < b``, ``a <= b``, ``a >= b``, ``a > b``
  - ``a == b``, ``a != b``: don't optimize bytes == str
  - ``obj in seq``, ``obj not in seq``: for bytes, str, tuple ``seq``
  - ``not x``: int

* str: ``str + str``, ``str * int``
* bytes: ``bytes + bytes``, ``bytes * int``
* tuple: ``tuple + tuple``, ``tuple * int``
* str, bytes, tuple, list: ``obj[index]``, ``obj[a:b:c]``
* dict: ``obj[index]``
* replace ``x in list`` with ``x in tuple`` if list only contains constants
* replace ``x in set`` with ``x in frozenset`` if set only contains constants
* simplify tests:

===================  ===========================
Code                 Constant folding
===================  ===========================
not(x is y)          x is not y
not(x is not y)      x is y
not(obj in seq)      obj not in seq
not(obj not in seq)  obj in seq
===================  ===========================

Note: ``not (x == y)`` is not replaced with ``x != y`` because ``not
x.__eq__(y)`` can be different than ``x.__ne__(y)`` for deliberate reason Same
rationale for not replacing ``not(x < y)`` with ``x >= y``.  For example,
``math.nan`` overrides comparison operators to always return ``False``.

Examples of optimizations:

===================  ===========================
Code                 Constant folding
===================  ===========================
-(5)                 -5
+5                   5
x in [1, 2, 3]       x in (1, 2, 3)
x in {1, 2, 3}       x in frozenset({1, 2, 3})
'Python' * 2         'PythonPython'
3 * (5,)             (5, 5, 5)
'python2.7'[:-2]     'python2'
'P' in 'Python'      True
9 not in (1, 2, 3)   True
[5, 9, 20][1]        9
===================  ===========================

:ref:`Configuration option <fat-config>`: ``constant_folding``.

See also the :ref:`constant folding <const-fold>` optimization.


.. _fat-replace-builtin-constant:

Replace builtin constants
-------------------------

Replace ``__debug__`` constant with its value.

:ref:`Configuration option <fat-config>`: ``replace_builtin_constant``.


.. _fat-dead-code:

Dead code elimination
---------------------

Remove the dead code.

Examples:

+--------------------------+--------------------------+
| Code                     | Dead code removed        |
+==========================+==========================+
| ::                       | ::                       |
|                          |                          |
|  if test:                |  if not test:            |
|      pass                |      else_block          |
|  else:                   |                          |
|      else_block          |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  if 1:                   |  body_block              |
|      body_block          |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  if 0:                   |  pass                    |
|      body_block          |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  if False:               |  else_block              |
|      body_block          |                          |
|  else:                   |                          |
|      else_block          |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  while 0:                |  pass                    |
|      body_block          |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  while 0:                |  else_block              |
|      body_block          |                          |
|  else:                   |                          |
|      else_block          |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  ...                     |  ...                     |
|  return ...              |  return ...              |
|  dead_code_block         |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  ...                     |  ...                     |
|  raise ...               |  raise ...               |
|  dead_code_block         |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  try:                    |  pass                    |
|      pass                |                          |
|  except ...:             |                          |
|      ...                 |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  try:                    |  else_block              |
|      pass                |                          |
|  except ...:             |                          |
|      ...                 |                          |
|  else:                   |                          |
|      else_block          |                          |
+--------------------------+--------------------------+
| ::                       | ::                       |
|                          |                          |
|  try:                    |  try:                    |
|      pass                |     else_block           |
|  except ...:             |  finally:                |
|      ...                 |     final_block          |
|  else:                   |                          |
|      else_block          |                          |
|  finally:                |                          |
|      final_block         |                          |
+--------------------------+--------------------------+

.. note::
   If a code block contains ``continue``, ``global``, ``nonlocal``, ``yield``
   or ``yield from``, it is not removed.

:ref:`Configuration option <fat-config>`: ``remove_dead_code``.

See also :ref:`dead code elimination <dead-code>` optimization.


.. _fat-copy-builtin-to-constant:

Copy builtin functions to constants
-----------------------------------

Opt-in optimization (disabled by default) to copy builtin functions to
constants.

Example with a function simple::

    def log(message):
        print(message)

+--------------------------------------------------+----------------------------------------------------+
| Bytecode                                         | Specialized bytecode                               |
+==================================================+====================================================+
| ::                                               | ::                                                 |
|                                                  |                                                    |
|   LOAD_GLOBAL   0 (print)                        |   LOAD_CONST      1 (<built-in function print>)    |
|   LOAD_FAST     0 (message)                      |   LOAD_FAST       0 (message)                      |
|   CALL_FUNCTION 1 (1 positional, 0 keyword pair) |   CALL_FUNCTION   1 (1 positional, 0 keyword pair) |
|   POP_TOP                                        |   POP_TOP                                          |
|   LOAD_CONST    0 (None)                         |   LOAD_CONST      0 (None)                         |
|   RETURN_VALUE                                   |   RETURN_VALUE                                     |
+--------------------------------------------------+----------------------------------------------------+

The first ``LOAD_GLOBAL`` instruction is replaced with ``LOAD_CONST``.
``LOAD_GLOBAL`` requires to lookup in the global namespace and then in the
builtin namespaces, two dictionary lookups. ``LOAD_CONST`` gets the value from
a C array, O(1) lookup.

The specialized bytecode requires two :ref:`guards <fat-guard>`:

* ``print`` builtin variable
* ``print`` global variable

The ``print()`` function is injected in the constants with the
``func.patch_constants()`` method.

The optimization on the builtin ``NAME`` requires two guards:

* ``NAME`` key in builtin namespace
* ``NAME`` key in global namespace

This optimization is disabled by default because it changes the :ref:`Python
semantics <fat-python-semantics>`: if the copied builtin function is replaced
in the middle of the function, the specialized bytecode still uses the old
builtin function. To use the optimization on a project, you may have to add the
following :ref:`configuration <fat-config>` at the top of the file::

    __astoptimizer__ = {'copy_builtin_to_constant': False}

:ref:`Configuration option <fat-config>`: ``copy_builtin_to_constant``.


See also:

* the :ref:`load globals and builtins when the module is loaded
  <load-global-optim>` optimization.
* `codetransformer <https://pypi.python.org/pypi/codetransformer>`_:
  ``@asconstants(len=len)`` decorator replaces lookups to the ``len`` name
  with the builtin ``len()`` function
* Thread on python-ideas mailing list: `Specifying constants for functions
  <https://mail.python.org/pipermail/python-ideas/2015-October/037028.html>`_
  by Serhiy Storchaka, propose to add ``const len=len`` (or alternatives)
  to declare a constant (and indirectly copy a builtin functions to constants)


.. _fat-simplify-iterable:

Simplify iterable
-----------------

Try to replace literals built at runtime with constants. Replace also
range(start, stop, step) with a tuple if the range fits in the
:ref:`configuration <fat-config>`.

When ``range(n)`` is replaced, two guards are required on ``range`` in builtin
and global namespaces and the function is specialized.

This optimization helps :ref:`loop unrolling <fat-loop-unroll>`.

Examples:

===========================   ===========================
Code                          Simplified iterable
===========================   ===========================
``for x in range(3): ...``    ``for x in (0, 1, 2): ...``
``for x in {}: ...``          ``for x in (): ...``
``for x in [4, 5. 6]: ...``   ``for x in (4, 5, 6): ...``
===========================   ===========================

:ref:`Configuration option <fat-config>`: ``simplify_iterable``.

See also :ref:`constant folding <fat-const-fold>`.


.. _fat-config:

Configuration
=============

It is possible to configure the AST optimizer per module by setting
the ``__astoptimizer__`` variable. Configuration keys:

* ``enabled`` (``bool``): set to ``False`` to disable all optimization (default: true)

* ``constant_propagation`` (``bool``): enable :ref:`constant propagation <fat-const-prop>`
  optimization? (default: true)

* ``constant_folding`` (``bool``): enable :ref:`constant folding
  <fat-const-fold>` optimization? (default: true)

* ``copy_builtin_to_constant`` (``bool``): enable :ref:`copy builtin functions
  to constants <fat-copy-builtin-to-constant>` optimization? (default: false)

* ``remove_dead_code`` (``bool``): enable :ref:`dead code elimination
  <fat-dead-code>` optimization? (default: true)

* maximum size of constants:

  - ``max_bytes_len``: Maximum number of bytes of a text string (default: 128)
  - ``max_int_bits``: Maximum number of bits of an integer (default: 256)
  - ``max_str_len``: Maximum number of characters of a text string (default: 128)
  - ``max_seq_len``: Maximum length in number of items of a sequence like
    tuples (default: 32). It is only a preliminary check: ``max_constant_size``
    still applies for sequences.
  - ``max_constant_size``: Maximum size in bytes of other constants
    (default: 128 bytes), the size is computed with ``len(marshal.dumps(obj))``

* ``replace_builtin_constant`` (``bool``): enable :ref:`replace builtin
  constants <fat-replace-builtin-constant>` optimization? (default: true)

* ``simplify_iterable`` (``bool``): enable :ref:`simplify iterable optimization
  <fat-simplify-iterable>`? (default: true)

* ``unroll_loops``: Maximum number of loop iteration for loop unrolling
  (default: ``16``). Set it to ``0`` to disable loop unrolling. See
  :ref:`loop unrolling <fat-loop-unroll>` optimization.

Example to disable all optimizations in a module::

    __astoptimizer__ = {'enabled': False}

Example to disable the constant folding optimization::

    __astoptimizer__ = {'constant_folding': False}


Comparison with the peephole optimizer
======================================

The :ref:`CPython peephole optimizer <cpython-peephole>` only implements a few
optimizations: :ref:`constant folding <const-fold>` and :ref:`dead code
elimination <dead-code>`. FAT Python implements more :ref:`optimizations
<fat-optim>`.

The peephole optimizer doesn't support :ref:`constant propagation
<fat-const-prop>`. Example::

    def f():
        x = 333
        return x

+----------------------------------+------------------------------------+
| Regular bytecode                 | FAT mode bytecode                  |
+==================================+====================================+
| ::                               | ::                                 |
|                                  |                                    |
|   LOAD_CONST               1 (1) |   LOAD_CONST               1 (333) |
|   STORE_FAST               0 (x) |   STORE_FAST               0 (x)   |
|   LOAD_FAST                0 (x) |   LOAD_CONST               1 (333) |
|   RETURN_VALUE                   |   RETURN_VALUE                     |
|                                  |                                    |
|                                  |                                    |
+----------------------------------+------------------------------------+

The :ref:`constant folding optimization <const-fold>` of the peephole optimizer
keeps original constants. For example, ``"x" + "y"`` is replaced with ``"xy"``
but ``"x"`` and ``"y"`` are kept. Example::

    def f():
        return "x" + "y"

+-----------------------------+------------------------+
| Regular constants           | FAT mode constants     |
+=============================+========================+
| ``(None, 'x', 'y', 'xy')``: | ``(None, 'xy')``:      |
| 4 constants                 | 2 constants            |
+-----------------------------+------------------------+

The peephole optimizer has a similar limitation even when building tuple
constants. The compiler produces AST nodes of type ``ast.Tuple``, the tuple
items are kept in code constants.


Limitations and Python semantic
===============================

FAT Python bets that the Python code is not modified when modules are loaded,
but only later, when functions and classes are executed. If this assumption is
wrong, FAT Python changes the semantics of Python.

.. _fat-python-semantics:

Python semantics
----------------

It is very hard, to not say impossible, to implementation and keep the exact
behaviour of regular CPython. CPython implementation is used as the Python
"standard". Since CPython is the most popular implementation, a Python
implementation must do its best to mimic CPython behaviour. We will call it the
Python semantics.

FAT Python should not change the Python semantics with the default
configuration.  Optimizations modifting the Python semantics must be disabled
by default: opt-in options.

As written above, it's really hard to mimic exactly CPython behaviour. For
example, in CPython, it's technically possible to modify local variables of a
function from anywhere, a function can modify its caller, or a thread B can
modify a thread A (just for fun). See :ref:`Everything in Python is mutable
<mutable>` for more information. It's also hard to support all introspections
features like ``locals()`` (``vars()``, ``dir()``), ``globals()`` and
``sys._getframe()``.

Builtin functions replaced in the middle of a function
------------------------------------------------------

FAT Python uses :ref:`guards <fat-guard>` to disable specialized function when
assumptions made to optimize the function are no more true. The problem is that
guard are only called at the entry of a function. For example, if a specialized
function ensures that the builtin function ``chr()`` was not modified, but
``chr()`` is modified during the call of the function, the specialized function
will continue to call the old ``chr()`` function.

The :ref:`copy builtin functions to constants <fat-copy-builtin-to-constant>`
optimization changes the Python semantics. If a builtin function is replaced
while the specialized function is optimized, the specialized function will
continue to use the old builtin function. For this reason, the optimization
is disabled by default.

Example::

    def func(arg):
        x = chr(arg)

        with unittest.mock.patch('builtins.chr', result='mock'):
            y = chr(arg)

        return (x == y)

If the :ref:`copy builtin functions to constants
<fat-copy-builtin-to-constant>` optimization is used on this function, the
specialized function returns ``True``, whereas the original function returns
``False``.

It is possible to work around this limitation by adding the following
:ref:`configuration <fat-config>` at the top of the file::

    __astoptimizer__ = {'copy_builtin_to_constant': False}

But the following use cases works as expected in FAT mode::

    import unittest.mock

    def func():
        return chr(65)

    def test():
        print(func())
        with unittest.mock.patch('builtins.chr', return_value="mock"):
            print(func())

Output::

    A
    mock

The ``test()`` function doesn't use the builtin ``chr()`` function.
The ``func()`` function checks its guard on the builtin ``chr()`` function only
when it's called, so it doesn't use the specialized function when ``chr()``
is mocked.


Guards on builtin functions
---------------------------

When a function is specialized, the specialization is ignored if a builtin
function was replaced after the end of the Python initialization. Typically,
the end of the Python initialization occurs just after the execution of the
``site`` module. It means that if a builtin is replaced during Python
initialization, a function will be specialized even if the builtin is not the
expected builtin function.

Example::

    import builtins

    builtins.chr = lambda: mock

    def func():
        return len("abc")

In this example, the ``func()`` is optimized, but the function is *not*
specialize. The internal call to ``func.specialize()`` is ignored because the
``chr()`` function was replaced after the end of the Python initialization.


Guards on type dictionary and global namespace
-----------------------------------------------

For other guards on dictionaries (type dictionary, global namespace), the guard
uses the current value of the mapping. It doesn't check if the dictionary value
was "modified".


Tracing and profiling
---------------------

Tracing and profiling works in FAT mode, but the exact control flow and traces
are different in regular and FAT mode. For example, :ref:`loop unrolling
<fat-loop-unroll>` removes the call to ``range(n)``.

See ``sys.settrace()`` and ``sys.setprofiling()`` functions.

Expected limitations
--------------------

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

Limitations of the AST optimizer
--------------------------------

See :ref:`Limitations of the AST optimizer <new-ast-optimizer-limits>`.


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


.. _fat-guard:

Guards
======

Guards:

* FuncGuard: check if a function was modified (currently only __code__ is
  checked)
* DictGuard: check if a dictionary key is created (if it didn't exist) or
  modified
* ArgTypeGuard: check the type of function arguments

Example: Guard on a builtin function
------------------------------------

Example of function::

    def use_builtin_len():
        return len("abc")

To replace ``len("abc")``, we have to ensure that:

* the builtin ``len()`` function was not overriden
  with ``builtins.len = mock_len``
* the ``len`` symbol was not added to the function globals which are the module
  globals

Example: Guard to inline a function
-----------------------------------

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


Implementation
==============

Steps and stages
----------------

The optimizer is splitted into multiple steps. Each optimization has its own
step: astoptimizer.const_fold.ConstantFolding implements for example constant
folding.

The function optimizer is splitted into two stages:

* stage 1: run steps which don't require function specialization
* stage 2: run steps which can add guard and specialize the function

Main classes:

* ModuleOptimizer: Optimizer for ast.Module nodes. It starts by looking for
  :ref:`__astoptimizer__ configuration <fat-config>`.
* FunctionOptimizer: Optimizer for ast.FunctionDef nodes. It starts by running
  FunctionOptimizerStage1.
* Optimizer: Optimizer for other AST nodes.

Steps used by ModuleOptimizer, Optimizer and FunctionOptimizerStage1:

* NamespaceStep: populate a Namespace object which tracks the local variables,
  used by ConstantPropagation
* ReplaceBuiltinConstant: replace builtin optimization
* ConstantPropagation: constant propagation optimization
* ConstantFolding: constant folding optimization
* RemoveDeadCode: dead code elimitation optimization

Steps used by FunctionOptimizer:

* NamespaceStep: populate a Namespace object which tracks the local variables
* UnrollStep: loop unrolling optimization
* CallPureBuiltin: call builtin optimization
* CopyBuiltinToConstantStep: copy builtins to constants optimization

Some optimizations produce a new AST tree which must be optimized again. For
example, loop unrolling produces new nodes like "i = 0" and duplicates the loop
body which uses "i". We need to rerun the optimizer on this new AST tree to run
optimizations like constant propagation or constant folding.


Files
-----

FAT python:

* Object/dictobject.c: add __version__
* Modules/fat.c: specialized functions with guards
* Tests

  - Lib/test/test_fat.py
  - Lib/test/fattester.py
  - Lib/test/fattesterast.py
  - Lib/test/fattesterast2.py

Other changes:

* Python/ceval.c: bugfixes when builtins is not a dict type
* Python/sysmodule.c: add sys.flags.fat
* Modules/main.c: add -F command line option

See also the :ref:`AST optimizer <new-ast-optimizer>`.


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


FAT Python API
==============

* func.specialize(bytecode[, guards: list]): add a specialized bytecode.
  If bytecode is a function, uses its __code__ attribute.
  Guards a list of dict, syntax of one guard:

  - ``{'guard_type': 'func', 'func': func2}``:
    guard on func2.__code__
  - ``{'guard_type': 'dict', 'dict': ns, 'keys': (key,)}``:
    guard on the versionned dictionary ns[key]
  - ``{'guard_type': 'builtins', 'names': ('len',)}``:
    guard on builtins.len (``builtins.__dict__['len']``) and
    ``globals()['len']``. The specialization is ignored if
    builtins.__dict__['len'] was replaced after the end of Python
    initialization or if globals()['len'] already exists.
  - ``{'guard_type': 'globals', 'names': ('obj',)}``:
    guard on globals()['obj']
  - ``{'guard_type': 'type_dict', 'type': MyClass, 'keys': ('attr',)}``:
    guard on MyClass.attr (on ``MyClass.__dict__['attr']``)
  - ``{'guard_type': 'arg_type', 'arg_index': 0, 'arg_types': (str,)}``:
    type of the function argument 0 must be ``str``.

* func.get_specialized()

For dictionary and function guards: specialized functions are removed if the
guards fail:

* Broken weak-reference to the dictionary/function
* The dictionary key was modified (created, modified or removed depending on
  the initial state)
* The function was modified
* An error occurred when getting the dictionary entry to get the key version


Origins of FAT Python
=====================

* :ref:`Old AST optimizer project <old-ast-optimizer>`
* :ref:`read-only Python <readonly>`
* Dave Malcolm wrote a patch modifying Python/eval.c to support specialized
  functions. See the http://bugs.python.org/issue10399
