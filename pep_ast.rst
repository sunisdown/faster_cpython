.. _pep-ast:

+++++++++++++++++++++++++++++
PEP: API for AST transformers
+++++++++++++++++++++++++++++

:ref:`FAT Python <fat-python>` PEPs:

* PEP 509: :ref:`Add a private version to dict <pep-dict-version>`
* PEP 510: :ref:`Specialized functions with guards <pep-specialize>`
* PEP xxx: :ref:`API for AST transformers <pep-ast>`

.. warning::
   This PEP is a draft, please wait until it's published on python-ideas
   or python-dev to discuss it. Or contact me privately.

::

    PEP: xxx
    Title: API for AST transformers
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

Propose an API to support AST transformers.


Rationale
=========

Python does not provide a standard way to transform the code. Projects
transforming the code use various hooks. The MacroPy project uses an
import hook: it adds its own module finder in ``sys.meta_path`` to
hook its AST transformer. Another option is to monkey-patch the
builtin ``compile()`` function. There are even more options to
hook a code transformer.

Transforming the code allows to extend the Python language for specific
needs. Transforming an Abstract Syntax Tree (AST) is a convenient way to
implement an optimizer. It's easier to work on the AST than working on
the bytecode, AST contains more information and is more high level.

Python 3.6 optimizes the code using a peephole optimizer. By
definition, a peephole optimizer has a narrow view of the code and so
can only implement basic optimizations. The optimizer rewrites the
bytecode. It is difficult to enhance because it written in C.

This PEP proposes to add an API to register AST transformers.

It will be possible to build a package of the transformed code: the
transformation is done ahead of time. It allows to implement powerful
but expensive transformations.

A new ``-o OPTIM_TAG`` command line option can be used to only load
transformed code: it changes the name of searched ``.pyc`` files.


Use Cases
=========

Interactive interpreter
-----------------------

It will be possible to use AST transformers with the interactive
interpreter. It is popular in Python and commonly used to demonstrate
Python.

Build a transformed package
---------------------------

It will be possible to build a package of the transformed code.

The package filename must be different to be able to install the
original package.

A transformer can have a configuration. The configuration is not stored
in the package. It is not possible to build two flavors of a package
with two different configurations with different filenames. Only one
package per combination of AST transformers can be build.

All ``.pyc`` files of the package must be transformed with the same AST
transformers and the same transformers configuration.


Install a transformed package
-----------------------------

It will be possible to install a package with specific
transformations. For example, install the optimized package or install
the regular package.


Run a transformed package
-------------------------

It will be possible to run a transformed package.

If a ``.pyc`` is missing and the required AST transformers are
available, Python creates the missing ``.pyc`` files on demand.

If a ``.pyc`` is missing and at least one required AST transformer is
missing, Python fails with an ``ImportError``. ``.py`` files are not
used. ``.pyc`` files are not written nor modified.


Changes
=======

API to support AST transformers:

* Add ``sys.ast_transformers``: list of callable with the prototype
  ``def ast_transformer(tree, filename)`` used to rewrite an AST tree.
  The list of empty by default (no AST transformer). The transformer is
  called after the creation of the AST by the parser and before the
  compilation to bytecode. It must return an AST tree. It can modify the
  AST tree in place, or create a new AST tree.
* Add ``sys.implementation.ast_transformers``: name of registered AST
  transformers
* Add ``sys.implementation.optim_tag``: optimization tag. It changes the
  filename of ``.pyc`` filename. Example:

  - ``Lib/__pycache__/os.cpython-36.pyc``: default filename
  - ``Lib/__pycache__/os.cpython-36.fat-0.pyc``: with the optimization
    tag ``"fat"``

* Add a new ``-o OPTIM_TAG`` command line option to set
  ``sys.implementation.optim_tag``

AST transformer changes:

* Add a new compiler flag ``PyCF_TRANSFORMED_AST`` to get the
  transformed AST. ``PyCF_ONLY_AST`` returns the AST before the
  transformers.
* Add ``ast.Constant``: this type is not emited by the compiler, but
  can be used in an AST transformer to simplify the code. It does not
  contain line number and column offset informations on tuple or
  frozenset items.
* ``PyCodeObject.co_lnotab``: line number delta becomes signed to support
  moving instructions (note: need to modify MAGIC_NUMBER in importlib).
* Enhance the compiler to support ``tuple`` and ``frozenset`` constants.
  Currently, ``tuple`` and ``frozenset`` constants are created by the
  peephole transformer, after the bytecode compilation.
* ``marshal`` module: fix serialization of the empty frozenset singleton
* update ``Tools/parser/unparse.py`` to support the new ``ast.Constant``
  node type


Example
=======

Amazing AST transformer replacing all strings with ``"Ni! Ni! Ni!"``::

    import ast
    import sys


    class KnightsWhoSayNi(ast.NodeTransformer):
        def visit_Str(self, node):
            node.s = 'Ni! Ni! Ni!'
            return node


    def ast_transformer(tree, filename):
        KnightsWhoSayNi().visit(tree)
        return tree

    # first register the AST transformer
    sys.ast_transformers.append(ast_transformer)
    sys.implementation.ast_transformers.append('knights_who_say_ni')

    # then update optimizer tag (used to build .pyc filenames)
    sys.implementation.optim_tag = 'knights_who_say_ni'

    # run code which will be transformed by ast_transformer()
    exec("print('Hello World!')")

Output::

    Ni! Ni! Ni!


Prior Art
=========

AST optimizers
--------------

In 2011, Eugene Toder proposes to rewrite some peephole optimizations in
a new AST optimizer: issue #11549, `Build-out an AST optimizer, moving
some functionality out of the peephole optimizer
<https://bugs.python.org/issue11549>`_.  The patch adds ``ast.Lit`` (it
was proposed to rename it to ``ast.Literal``).

In 2012, Victor Stinner wrote the `astoptimizer
<https://bitbucket.org/haypo/astoptimizer/>`_ project, an AST optimizer
implementing various optimizations. Most interesting optimizations break the
Python semantics since no guard is used to disable optimization if something
changes.

Issue #17515: `Add sys.setasthook() to allow to use a custom AST
optimizer <https://bugs.python.org/issue17515>`_.


Python Preprocessors
--------------------

* `MacroPy <https://github.com/lihaoyi/macropy>`_: MacroPy is an
  implementation of Syntactic Macros in the Python Programming Language.
  MacroPy provides a mechanism for user-defined functions (macros) to
  perform transformations on the abstract syntax tree (AST) of a Python
  program at import time.
* `pypreprocessor <https://code.google.com/p/pypreprocessor/>`_: C-style
  preprocessor directives in Python, like ``#define`` and ``#ifdef``


Modify the bytecode
-------------------

* `codetransformer <https://pypi.python.org/pypi/codetransformer>`_:
  Bytecode transformers for CPython inspired by the ``ast`` module’s
  ``NodeTransformer``.
* `byteplay <http://code.google.com/p/byteplay/>`_: Byteplay lets you
  convert Python code objects into equivalent objects which are easy to
  play with, and lets you convert those objects back into living Python
  code objects. It's useful for applying crazy transformations on Python
  functions, and is also useful in learning Python byte code
  intricacies. See `byteplay documentation
  <http://wiki.python.org/moin/ByteplayDoc>`_.

See also `BytecodeAssembler <http://pypi.python.org/pypi/BytecodeAssembler>`_.


Copyright
=========

This document has been placed in the public domain.
