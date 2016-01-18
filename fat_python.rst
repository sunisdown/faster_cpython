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

See the `fatoptimizer documentation <https://fatoptimizer.readthedocs.org/>`_
which is the main part of FAT Python.

The FAT Python project is made of multiple parts:

* The `fatoptimizer project <https://fatoptimizer.readthedocs.org/>`_ is the
  static optimizer for Python 3.6 using function specialization with guards. It
  is implemented as an AST optimizer.
* The `fat module <https://fatoptimizer.readthedocs.org/en/latest/fat.html>`_
  is Python extension module (written in C) implementing fast guards. The
  ``fatoptimizer`` optimizer uses ``fat`` guards to only use the specialize
  bytecode under some conditions.
* Patches for Python 3.6:

  * `PEP 509: Add ma_version to PyDictObject
    <https://bugs.python.org/issue26058>`_
  * `PEP 510: Specialize functions with guards
    <https://bugs.python.org/issue26098>`_
  * PEP 511 patches:

    * `PEP 511: Add test.support.optim_args_from_interpreter_flags()
      <https://bugs.python.org/issue26100>`_
    * `PEP 511: code.co_lnotab: use signed line number delta to support moving
      instructions in an optimizer
      <https://bugs.python.org/issue26107>`_
    * `PEP 511: Add sys.set_code_transformers()
      <http://bugs.python.org/issue26145>`_
    * ast.Constant patch (not available yet)

  * Somehow related to the PEP 511:

    * `Lib/test/test_compileall.py fails when run directly
      <http://bugs.python.org/issue26101>`_
    * `site ignores ImportError when running sitecustomize and usercustomize
      <http://bugs.python.org/issue26099>`_

* Python Enhancement Proposals (PEP):

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


Test FAT Python
===============

Download FAT Python with::

    hg clone http://hg.python.org/sandbox/fatpython

Compile it::

    ./configure && make

Run the full Python test suite::

    ./python -X fat -m test -j0

To use specialized code without registered fatoptimizer, first you
have to compile (and optimized) the stdlib::

    ./python -X fat -m compileall

Then you can use the optimized stdlib without fatoptimizer::

    ./python -o fat
    # enjoy!


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

    >>> import fat
    >>> len(fat.get_specialized(func))
    1
    >>> specialized_code = fat.get_specialized(func)[0][0]
    >>> dis.dis(specialized_code['code'])
      2           0 LOAD_CONST               1 (3)
                  3 RETURN_VALUE

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


Origins of FAT Python
=====================

* :ref:`Old AST optimizer project <old-ast-optimizer>`
* :ref:`read-only Python <readonly>`
* Dave Malcolm wrote a patch modifying Python/eval.c to support specialized
  functions. See the http://bugs.python.org/issue10399
