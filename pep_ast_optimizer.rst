.. _pep-ast-optimizer:

++++++++++++++++++++++
PEP: AST optimizer API
++++++++++++++++++++++

FAT Python PEPs:

* PEP 1/3: :ref:`dict.__version__ <pep-dict-version>`
* PEP 2/3: :ref:`AST optimizer API <pep-ast-optimizer>`
* PEP 3/3: :ref:`FAT mode, specialized bytecode with guards <pep-fat-mode>`

.. warning::
   This PEP is a draft, please wait until it's published on python-ideas
   or python-dev to discuss it. Or contact me privately.

::

    PEP: xxx
    Title: API for AST optimizers
    Version: $Revision$
    Last-Modified: $Date$
    Author: Victor Stinner <victor.stinner@gmail.com>
    Status: Draft
    Type: Standards Track
    Content-Type: text/x-rst
    Created: 4-January-2016
    Python-Version: 3.6


Abstract
========

Propose an API to support AST optimizers.


Rationale
=========

CPython 3.5 optimizes the code using a peephole optimizer. The optimizer
rewrites the bytecode, is written in C and is difficult to enhance. By
definition, the optimizer is limited because it has a narrow view of the
code.

Working at AST level is simpler. For example, it's easy to match a
pattern with a new AST node.

This PEP proposes changes, especially the addition of ``sys.astoptimizer``,
to support AST optimizers.


Changes
=======

Main changes:

* Add ``sys.astoptimizer``: callable with prototype
  ``def optimizer(tree, filename)`` used to rewrite an AST tree,
  ``None`` by default (not used).
* Add a new compiler flag ``PyCF_OPTIMIZED_AST`` to get the optimized
  AST, ``PyCF_ONLY_AST`` returns the AST before the optimizer.
* Add ``ast.Constant``: this type is not emited by the compiler, but
  only used internally in an AST optimizer to simplify the code. It
  doesn't contain line number and column offset informations on tuple or
  frozenset items.
* PyCodeObject.co_lnotab: line number delta becomes signed to support
  moving instructions => need to modify MAGIC_NUMBER in importlib

Implementation:

* Enhance compiler to emit correctly constants
* marshal: fix serialization of the empty frozenset singleton
* update Tools/parser/unparse.py for ast.Constant


Prior Art
=========

In 2011, Eugene Toder proposes to rewrite the peephole optimizer with
an AST optimizer: issue #11549, `Build-out an AST optimizer, moving some functionality
out of the peephole optimizer <https://bugs.python.org/issue11549>`_.
The patch adds ``ast.Lit`` (it was proposed to renamed it to ``ast.Literal``).

Issue #17515: `Add sys.setasthook() to allow to use a custom AST
optimizer <https://bugs.python.org/issue17515>`_.


Copyright
=========

This document has been placed in the public domain.
