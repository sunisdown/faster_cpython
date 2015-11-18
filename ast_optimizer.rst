*************
AST Optimizer
*************

Intro
=====

An AST optimizer rewrites the Abstract Syntax Tree (AST) of a Python module to
produce a more efficient code. Currently in CPython 3.5, only basic
optimizations are implemented by rewriting the bytecode. The optimizer is
called the "peepholer" and is written in the C language.

Old AST optimizer project
=========================

https://bitbucket.org/haypo/astoptimizer/ was a first attempt to optimize
Python. This project was rejected by the Python community because it breaks the
Python semantic. For example, it replaces ``len("abc")`` with ``3``. It checks
that ``len()`` was not overriden in the module, but it doesn't check that the
builtin ``len()`` function was not overriden.

New AST optimizer combined with FAT Python
==========================================

The :ref:`FAT Python <fatpython>` project provides guards checks at runtime. It
comes with an AST optimizer which produces guards to disable optimizations if
any assumption is no more true.


Links
=====

CPython issues
--------------

* `Issue #1346238 <http://bugs.python.org/issue1346238>`_:
  A constant folding optimization pass for the AST
* `Issue #2181 <http://bugs.python.org/issue2181>`_:
  optimize out local variables at end of function
* `Issue #2499 <http://bugs.python.org/issue2499>`_:
  Fold unary + and not on constants
* `Issue #4264 <http://bugs.python.org/issue4264>`_:
  Patch: optimize code to use LIST_APPEND instead of calling list.append
* `Issue #7682 <http://bugs.python.org/issue7682>`_:
  Optimisation of if with constant expression
* `Issue #10399 <http://bugs.python.org/issue10399>`_:
  AST Optimization: inlining of function calls
* `Issue #11549 <http://bugs.python.org/issue11549>`_:
  Build-out an AST optimizer, moving some functionality out of the peephole optimizer
* `Issue #17068 <http://bugs.python.org/issue17068>`_:
  peephole optimization for constant strings
* `Issue #17430 <http://bugs.python.org/issue17430>`_:
  missed peephole optimization

AST
---

* `instrumenting_the_ast.html <http://www.dalkescientific.com/writings/diary/archive/2010/02/22/instrumenting_the_ast.html>`_
* `the-internals-of-python-generator-functions-in-the-ast
  <http://tomlee.co/2008/04/the-internals-of-python-generator-functions-in-the-ast/>`_
* `tlee-ast-optimize branch
  <http://svn.python.org/view/python/branches/tlee-ast-optimize/Python/optimize.c?view=log>`_
* `ast-optimization-branch-elimination-in-generator-functions
  <http://grokbase.com/p/python/python-dev/0853rf4s1a/ast-optimization-branch-elimination-in-generator-functions>`_

Bytecode
--------

* `byteplay <http://code.google.com/p/byteplay/>`_
* `diving-into-byte-code-optimization-in-python
  <http://www.slideshare.net/cjgiridhar/diving-into-byte-code-optimization-in-python>`_
* `BytecodeAssembler <http://pypi.python.org/pypi/BytecodeAssembler>`_
* `ByteplayDoc <http://wiki.python.org/moin/ByteplayDoc>`_
* `Hacking Python bytecode <http://geofft.mit.edu/blog/sipb/73>`_

