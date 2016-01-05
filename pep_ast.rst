.. _pep-ast-optimizer:

++++++++++++++++++++++
PEP: AST optimizer API
++++++++++++++++++++++

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


Changes
=======

Main changes:

* Add ``sys.asthook``: callable with prototype
  ``def hook(tree, filename)`` used to rewrite an AST tree, ``None`` by
  default (not used).
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


Alternatives
============

Add a new flag, similar to ``PyCF_ONLY_AST``, to get AST without the AST
hook.


Prior Art
=========

The issue #11549 `Build-out an AST optimizer, moving some functionality
out of the peephole optimizer <https://bugs.python.org/issue11549>`_
proposes to add ``ast.Lit`` (or ``ast.Literal``).

Issue #17515: `Add sys.setasthook() to allow to use a custom AST
optimizer <https://bugs.python.org/issue17515>`_.


Copyright
=========

This document has been placed in the public domain.
