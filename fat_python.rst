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
  is a Python extension module (written in C) implementing fast guards. The
  ``fatoptimizer`` optimizer uses ``fat`` guards to specialize functions.
  ``fat`` guards are used to verify assumptions used to specialize the code. If
  an assumption is no more true, the specialized code is not used. The ``fat``
  module is required to run code optimized by ``fatoptimizer`` if at least one
  function is specialized.
* Python Enhancement Proposals (PEP):

  * PEP 509: `Add a private version to dict
    <https://www.python.org/dev/peps/pep-0509/>`_
  * PEP 510: `Specialized functions with guards
    <https://www.python.org/dev/peps/pep-0510/>`_
  * PEP 511: `API for AST transformers
    <https://www.python.org/dev/peps/pep-0511/>`_

* Patches for Python 3.6:

  * `PEP 509: Add ma_version to PyDictObject
    <https://bugs.python.org/issue26058>`_
  * `PEP 510: Specialize functions with guards
    <https://bugs.python.org/issue26098>`_
  * `PEP 511: Add sys.set_code_transformers()
    <http://bugs.python.org/issue26145>`_
  * Related to the PEP 511:

    * *DONE*: `PEP 511: Add test.support.optim_args_from_interpreter_flags()
      <https://bugs.python.org/issue26100>`_
    * *DONE*: `PEP 511: code.co_lnotab: use signed line number delta to support moving
      instructions in an optimizer
      <https://bugs.python.org/issue26107>`_
    * *DONE*: `PEP 511: Add ast.Constant to allow AST optimizer to emit constants
      <http://bugs.python.org/issue26146>`_
    * *DONE*: `Lib/test/test_compileall.py fails when run directly
      <http://bugs.python.org/issue26101>`_
    * *DONE*: `site ignores ImportError when running sitecustomize and usercustomize
      <http://bugs.python.org/issue26099>`_
    * *DONE*: `code_richcompare() don't use constant type when comparing code constants
      <http://bugs.python.org/issue25843>`_

Announcements and status reports:

* `Status of the FAT Python project, January 12, 2016
  <https://haypo.github.io/fat-python-status-janv12-2016.html>`_
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


.. _fat-getting-starting:

Getting started
===============

Compile Python 3.6 patched with PEP 509, PEP 510 and PEP 511::

    hg clone http://hg.python.org/sandbox/fatpython
    cd fatpython
    ./configure --with-pydebug CFLAGS="-O0" && make

Install fat::

    git clone https://github.com/haypo/fat
    cd fat
    ../python setup.py build
    cp -v build/lib*/fat.*so ../Lib
    cd ..

Install fatoptimizer::

    git clone https://github.com/haypo/fatoptimizer
    (cd Lib; ln -s ../fatoptimizer/fatoptimizer .)

``fatoptimizer`` is registed by the ``site`` module if ``-X fat`` command line
option is used. Extract of ``Lib/site.py``::

    if 'fat' in sys._xoptions:
        import fatoptimizer
        fatoptimizer._register()

Check that fatoptimizer is registered with::

    $ ./python -X fat -c 'import sys; print(sys.implementation.optim_tag)'
    fat-opt

You must get ``fat-opt`` (and not ``opt``).


How can you contribute?
=======================

The `fatoptimizer project <https://fatoptimizer.readthedocs.org/>`_ needs the
most love. Currently, the optimizer is not really smart. There is a long `TODO
list <https://fatoptimizer.readthedocs.org/en/latest/todo.html>`_. Pick a
simple optimization, try to implement it, send a pull request on GitHub. At
least, any kind of feedback is useful ;-)

If you know the C API of Python, you may also review the implementation of the
PEPs:

* `PEP 509: Add ma_version to PyDictObject
  <https://bugs.python.org/issue26058>`_
* `PEP 510: Specialize functions with guards
  <https://bugs.python.org/issue26098>`_
* `PEP 511: Add sys.set_code_transformers()
  <http://bugs.python.org/issue26145>`_

But these PEPs are still work-in-progress, so the implementation can still
change.


Play with FAT Python
====================

See :ref:`Getting started <fat-getting-starting>` to compile FAT Python.


Disable peephole optimizer
--------------------------

The ``-o noopt`` command line option disables the Python peephole optimizer::

    $ ./python -o noopt -c 'import dis; dis.dis(compile("1+1", "test", "exec"))'
      1           0 LOAD_CONST               0 (1)
                  3 LOAD_CONST               0 (1)
                  6 BINARY_ADD
                  7 POP_TOP
                  8 LOAD_CONST               1 (None)
                 11 RETURN_VALUE


Specialized code calling builtin function
-----------------------------------------

Test fatoptimizer on builtin function::

    $ ./python -X fat
    >>> def func(): return len("abc")
    ...

    >>> import dis
    >>> dis.dis(func)
      1           0 LOAD_GLOBAL              0 (len)
                  3 LOAD_CONST               1 ('abc')
                  6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
                  9 RETURN_VALUE

    >>> import fat
    >>> fat.get_specialized(func)
    [(<code object func at 0x7f9d3155b1e0, file "<stdin>", line 1>,
    [<fat.GuardBuiltins object at 0x7f9d39191198>])]

    >>> dis.dis(fat.get_specialized(func)[0][0])
      1           0 LOAD_CONST               1 (3)
                  3 RETURN_VALUE

The specialized code is removed when the function is called if the builtin
function is replaced (here by declaring a ``len()`` function in the global
namespace)::

    >>> len=lambda obj: "mock"
    >>> func()
    'mock'
    >>> fat.func_get_specialized(func)
    []


Microbenchmark
--------------

Run a microbenchmark on specialized code::

    $ ./python -m timeit -s 'def f(): return len("abc")' 'f()'
    10000000 loops, best of 3: 0.122 usec per loop

    $ ./python -X fat -m timeit -s 'def f(): return len("abc")' 'f()'
    10000000 loops, best of 3: 0.0932 usec per loop

Python must be optimized to run a benchmark: use ``./configure && make clean &&
make`` if you previsouly compiled it in debug mode.

You should compare specialized code to an unpatched Python 3.6 to run a fair
benchmark (to also measure the overhead of PEP 509, 510 and 511 patches).


Run optimized code without registering fatoptimizer
===================================================

You have to compile optimized .pyc files::

    # the optimizer is slow, so add -v to enable fatoptimizer logs for more fun
    ./python -X fat -v -m compileall

    # why does compileall not compile encodings/*.py?
    ./python -X fat -m py_compile Lib/encodings/{__init__,aliases,latin_1,utf_8}.py


Finally, enjoy optimized code with no registered optimized::

    $ ./python -o fat-opt -c 'import sys; print(sys.implementation.optim_tag, sys.get_code_transformers())'
    fat-opt []

Remember that you cannot import .py files in this case, only .pyc::

    $ echo 'print("Hello World!")' > hello.py
    $ ENV/bin/python -o fat-opt -c 'import hello'
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
    ImportError: missing AST transformers for 'hello.py': optim_tag='fat-opt', transformers tag='noopt'


Origins of FAT Python
=====================

* :ref:`Old AST optimizer project <old-ast-optimizer>`
* :ref:`read-only Python <readonly>`
* Dave Malcolm wrote a patch modifying Python/eval.c to support specialized
  functions. See the http://bugs.python.org/issue10399
