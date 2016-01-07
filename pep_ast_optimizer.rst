.. _pep-ast-optimizer:

++++++++++++++++++++++
PEP: AST optimizer API
++++++++++++++++++++++

FAT Python PEPs:

* PEP 1/3: :ref:`dict.__version__ <pep-dict-version>`
* PEP 2/3: :ref:`AST optimizer API <pep-ast-optimizer>`
* PEP 3/3: :ref:`Specialized bytecode with guards <pep-fat-mode>`

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

Propose an API to support AST optimizers and add a "FAT" mode.


Rationale
=========

CPython 3.5 optimizes the code using a peephole optimizer. By
definition, a peephole optimizer has a narrow view of the code and so
can only implement a few optimizations. The optimizer rewrites the
bytecode, is written in C and is difficult to enhance.

To keep the peephole simple and efficient, optimizations are bypassed if
the line number table (``code.co_lnotab``) is too complex (when a line
number delta is larger than 255) or if the bytecode is longer than 32700
bytes.

Only basic optimizations are implemented: constant folding optimizations
on jumps and dead code elimination. For example, `constant propagation
optimization <https://en.wikipedia.org/wiki/Copy_propagation>`_ is not
implemented whereas it only need a basic knownledge of the code.

Working on the `Abstract Syntax Tree` (AST) level is simpler. It is a
high-level abstraction which contains more information than bytecode.
For example, it's easy to match a pattern with a new AST node.

This PEP proposes to add an API to support pluggable AST optimizers.

Some optimizations like constant folding can be rewritten in an AST
optimizer. Even if most optimizations currently implemented in the
peephole optimizer can be reimplemented in an AST optimizer, the
peephole optimizer remains useful since some optimizations are specific
to the bytecode. For example, optimizations on jumps remains useful on
the bytecode.

Adding a default AST optimizer is out of the scope of the PEP. Including
a default AST optimizer to Python will require a separated PEP.

Optimizations more expensive than basic optimizations implemented in the
current peephole optimizer are expected. That's why a new "FAT mode" is
introduced.  Python .py files are compiled to .pyc when modules and
applications are installed, but for scripts, the bytecode is compiled at
runtime. Moreover, some optimizations may break the Python semantic in
subtle ways. Having to enable explicitly the FAT mode is required to say
"ok, I make compromises on the Python semantic for performance, I am
aware of the issues".


Changes
=======

"FAT" mode:

* Add a new ``-F`` command line option to enable FAT mode
* Add ``sys.flags.fat``
* ``importlib`` module: new filename for ``.pyc`` files in FAT mode. Example:

  - Lib/__pycache__/os.cpython-36.pyc: default mode
  - Lib/__pycache__/os.cpython-36.fat-0.pyc: FAT mode

Main changes:

* Add ``sys.astoptimizer``: callable with prototype
  ``def optimizer(tree, filename)`` used to rewrite an AST tree,
  ``None`` by default (not used). The optimizer is called after the
  creation of the AST and before the compilation to bytecode.
* Add a new compiler flag ``PyCF_OPTIMIZED_AST`` to get the optimized
  AST, ``PyCF_ONLY_AST`` returns the AST before the optimizer.
* Add ``ast.Constant``: this type is not emited by the compiler, but
  only used internally in an AST optimizer to simplify the code. It
  doesn't contain line number and column offset informations on tuple or
  frozenset items.
* ``PyCodeObject.co_lnotab``: line number delta becomes signed to support
  moving instructions => need to modify MAGIC_NUMBER in importlib

Implementation:

* Enhance the compiler to support ``tuple`` and ``frozenset`` constants.
  Currently, ``tuple`` and ``frozenset`` constants are created by the
  peephole optimizer, after the bytecode compilation.
* ``marshal`` module: fix serialization of the empty frozenset singleton
* update ``Tools/parser/unparse.py`` to support the new ``ast.Constant``
  node type


Example
=======

Optimizer replacing all strings with ``"Ni! Ni! Ni!"``::

    import ast
    import sys


    class Optimizer(ast.NodeTransformer):
        def visit_Str(self, node):
            node.s = 'Ni! Ni! Ni!'
            return node


    def optimizer(tree, filename):
        Optimizer().visit(tree)
        return tree


    sys.astoptimizer = optimizer
    exec("print('Hello World!')")

Output::

    Ni! Ni! Ni!


Prior Art
=========

In 2011, Eugene Toder proposes to rewrite some peephole optimizations in
a new AST optimizer: issue #11549, `Build-out an AST optimizer, moving
some functionality out of the peephole optimizer
<https://bugs.python.org/issue11549>`_.  The patch adds ``ast.Lit`` (it
was proposed to rename it to ``ast.Literal``).

Issue #17515: `Add sys.setasthook() to allow to use a custom AST
optimizer <https://bugs.python.org/issue17515>`_.

Previous attempts to implement AST optimizers were abandonned because
the speedup was negligible compared to the effort to implement them, or
because optimizations changed the Python semantic.

Supporting specialized bytecode with guards (PEP xxx) allow to implement
more efficient optimizations without breaking the Python semantic.
Adding a new ``dict.__version__`` property (PEP yyy) allows to implement
efficient guards on namespaces to check if a variable was replaced.


Copyright
=========

This document has been placed in the public domain.
