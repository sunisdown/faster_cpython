.. _ast-optimizers:

**************
AST Optimizers
**************

Intro
=====

An AST optimizer rewrites the Abstract Syntax Tree (AST) of a Python module to
produce a more efficient code. Currently in CPython 3.5, only basic
optimizations are implemented by rewriting the bytecode. The optimizer is
called the "peepholer" and is written in the C language.

Old AST optimizer project
=========================

See :ref:`old AST optimizer <old-ast-optimizer>`.


New AST optimizer combined with FAT Python
==========================================

See :ref:`new AST optimizer <new-ast-optimizer>`.


pythran AST
===========

pythran.analysis.PureFunctions of pythran project, depend on ArgumentEffects
and GlobalEffects analysis: automatically detect pure functions.


PyPy AST optimizer
==================

https://bitbucket.org/pypy/pypy/src/default/pypy/interpreter/astcompiler/optimize.py


Cython AST optimizer
====================

https://mail.python.org/pipermail/python-dev/2012-August/121300.html

* `Compiler/Optimize.py
  <https://github.com/cython/cython/blob/master/Cython/Compiler/Optimize.py>`_
* `Compiler/ParseTreeTransforms.py
  <https://github.com/cython/cython/blob/master/Cython/Compiler/ParseTreeTransforms.py>`_
* `Compiler/Builtin.py
  <https://github.com/cython/cython/blob/master/Cython/Compiler/Builtin.py>`_
* `Compiler/Pipeline.py
  <https://github.com/cython/cython/blob/master/Cython/Compiler/Pipeline.py#L123>`_


Links
=====

CPython issues
--------------

* `Issue #2181 <http://bugs.python.org/issue2181>`_:
  optimize out local variables at end of function
* `Issue #2499 <http://bugs.python.org/issue2499>`_:
  Fold unary + and not on constants
* `Issue #4264 <http://bugs.python.org/issue4264>`_:
  Patch: optimize code to use LIST_APPEND instead of calling list.append
* `Issue #7682 <http://bugs.python.org/issue7682>`_:
  Optimisation of if with constant expression
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
